---
name: Nanite Software Rasterizer (Compute)
description: 小三角形（数ピクセル以下）を compute シェーダで HW より高速にラスタするアルゴリズムと UE 実装
type: project
---

# Nanite Software Rasterizer（Compute シェーダラスタライザ）

- 上位: [[_algorithm_index]]
- 関連: [[nanite_cluster_culling]] / [[nanite_visibility_buffer]] / [[nanite_lod]]
- 採用システム: 1 三角形が概ね < 32 ピクセルとなる Nanite のラスタパス（多数派）
- 出典 ID: **S20**（[[_source_index]]）— §3 Software Rasterization
- 影響元: Olano & Greer 1997 "Triangle Scan Conversion using 2D Homogeneous Coordinates"（HW のエッジ関数定式化）

---

## 1. 何のためのアルゴリズムか

GPU の固定関数ラスタライザは:
- **大三角形では超高速**（数千年の最適化）
- **1 ピクセル未満の三角形では効率が悪い**（quad（2x2）単位で割当・3 ピクセルが unused）

Nanite は 1 ピクセル ≒ 1 三角形を狙うので、HW ラスタは効率が落ちる。

### 素朴な手法の問題

- **HW ラスタ単独**: 数ピクセル三角形で 50-75% の SIMD レーンが unused
- **完全 SW ラスタ**: 大三角形で HW に大きく劣る

### Nanite の貢献

- **辺長予測**で SW / HW を切替（`MinPixelsPerEdgeHW = 32`）
- **小三角形 → SW Rasterizer (compute)**: スレッドグループで頂点 cache + バリセントリック評価
- **大三角形 → HW Rasterizer (Mesh/Prim Shader)**: 固定関数の利点を活かす
- 共通の **VisBuffer64 + InterlockedMax** で結果統合

---

## 2. 理論

### 2.1 SW ラスタアルゴリズム（半空間テスト / Edge Function）

各三角形の 3 辺に対し edge function `E_i(x, y) = a*x + b*y + c` を計算。`E_0,1,2 ≥ 0` の点が三角形内部。整数演算（subpixel 16x16 = 256 単位）で誤差ゼロ。

```hlsl
// 擬似コード
for each pixel in BBox(triangle):
    if E0(p) >= 0 and E1(p) >= 0 and E2(p) >= 0:
        depth = barycentric * triangleDepth
        InterlockedMax(VisBuffer64, packed(depth, ID))
```

UE 実装は `RasterizeTri_Adaptive`（`NaniteRasterizer.usf:666` 等）で適応的に走査:
- 小三角形: 直接 BBox 走査
- 中三角形: 8x8 / 4x4 タイル単位で early reject

### 2.2 SW vs HW 切替の閾値

各クラスタの **画面投影サイズ → 平均辺長** を推定し、`r.Nanite.MinPixelsPerEdgeHW = 32` ピクセル以上ならクラスタを HW path に振り分け。`NaniteCullRaster.cpp:157-162` で定義。

```cpp
TAutoConsoleVariable<float> CVarNaniteMinPixelsPerEdgeHW(
    TEXT("r.Nanite.MinPixelsPerEdgeHW"),
    32.0f,
    TEXT("The triangle edge length in pixels at which Nanite starts using the hardware rasterizer."));
```

### 2.3 Subpixel Snap & Coverage

頂点座標は `NANITE_SUBPIXEL_SAMPLES = 256`（8.8 fixed point）にスナップ。`SetupTriangle< NANITE_SUBPIXEL_SAMPLES, !NANITE_TWO_SIDED >`（`NaniteRasterizer.usf:648, 740`）が:
- 頂点座標 → 整数 subpixel
- Edge function 係数計算
- BBox / Scissor クリップ
- Backface 判定（Two-Sided でなければ）

### 2.4 頂点キャッシュ（クラスタ単位）

128 三角形クラスタは典型 128 頂点未満を共有 → スレッドグループ shared memory に頂点を詰める:

```hlsl
// NaniteRasterizer.usf:684-717 より要約
for VertIndex < Cluster.NumVerts:
    InputVert = FetchAndDeformLocalNaniteVertex(...)
    PointTranslatedWorld = mul(InputVert.Position, LocalToTranslatedWorld) + WPO
    PointClip = mul(PointTranslatedWorld, TranslatedWorldToClip)
    GroupVerts[VertIndex] = CalculateSubpixelCoordinates(Raster, PointClip).xyz
GroupMemoryBarrierWithGroupSync()
```

その後 `GroupVerts[VertIndexes.x/y/z]` で三角形を組み立て。

### 2.5 Threadgroup サイズ

```hlsl
// NaniteRasterizer.usf:62-66
#if NANITE_VERT_REUSE_BATCH || NANITE_VOXELS
    #define THREADGROUP_SIZE 32
#else
    #define THREADGROUP_SIZE 64
#endif
```

Wave サイズ（NV: 32, AMD: 64）に合わせる。

### 2.6 InterlockedMax 競合

Visibility Buffer 書込:
```hlsl
uint PixelValue = (VisibleIndex + 1) << 7;
PixelValue |= TriIndex;
// VisBuffer64 は uint64 アトミック、上位 32-bit が depth、下位 32-bit が PixelValue
InterlockedMax(VisBuffer64[pixel], pack64(depth, PixelValue));
```

depth が大きい（近い）ものが勝つ（reverse-Z）。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `NaniteRasterizer.usf` | SW ラスタ本体（compute）+ HW 共通 PS |
| `NaniteRasterizationCommon.ush` | `RasterizeTri_Adaptive` / `SetupTriangle` / `CalculateSubpixelCoordinates` |
| `NaniteRasterBinning.usf` | クラスタを SW/HW ビンに振り分け |
| `NaniteCullRaster.cpp` | パスオーケストレーション |
| `NaniteVertexDeformation.ush` | スキニング/Spline/WPO 適用 |

### 3.2 SW Raster Bin Fetch（`NaniteRasterizer.usf:91-99`）

```hlsl
// .x = VisibleIndex
// .y = RangeStart
// .z = RangeEnd
// .w = MaterialFlags
uint4 FetchSWRasterBin(const uint ClusterIndex)
{
    const uint RasterBinOffset = RasterBinMeta[GetRasterBin()].ClusterOffset;
    const uint2 PackedData     = RasterBinData[RasterBinOffset + ClusterIndex].xy;
    const uint VisibleIndex    = PackedData.x;
    const uint RangeStart      = PackedData.y >> 16u;
    const uint RangeEnd        = PackedData.y & 0xFFFFu;
    return uint4(VisibleIndex, RangeStart, RangeEnd, RasterBinMeta[GetRasterBin()].MaterialFlags_DepthBlock & 0xFFFFu);
}
```

HW 用は `FetchHWRasterBin`（`NaniteRasterizer.usf:105-114`）— SW と HW のクラスタは同じバッファに **両端から書込** されており、`(BinSWCount + BinHWCount - 1) - ClusterIndex` で逆順アクセス。

### 3.3 三角形ラスタの本体（`NaniteRasterizer.usf:740-771`）

```hlsl
FRasterTri Tri = SetupTriangle< NANITE_SUBPIXEL_SAMPLES, !NANITE_TWO_SIDED >( Raster.ScissorRect, Verts );

if( Tri.bIsValid )
{
    uint PixelValue = (VisibleIndex + 1) << 7;
    PixelValue |= TriIndex;

    uint2 VisualizeValues = GetVisualizeValues();

    TNaniteWritePixel< FMaterialShader > NaniteWritePixel;
    NaniteWritePixel.Raster = Raster;
    NaniteWritePixel.Shader = MaterialShader;
    NaniteWritePixel.PixelValue = PixelValue;
    NaniteWritePixel.VisualizeValues = VisualizeValues;
    RasterizeTri_Adaptive( Tri, NaniteWritePixel );
}
```

`TNaniteWritePixel` テンプレートで通常 / VSM ページキャッシュ書込を切替。

### 3.4 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.Nanite.MinPixelsPerEdgeHW` | 32.0 | 辺長閾値（< なら SW） |
| `r.Nanite.ComputeRasterization` | 1 | SW ラスタ ON/OFF |
| `r.Nanite.MeshShaderRasterization` | 1 | HW: Mesh Shader 利用 |
| `r.Nanite.PrimShaderRasterization` | 1 | HW: Prim Shader 利用 |
| `r.Nanite.AsyncRasterization` | 1 | async compute で実行 |
| `r.Nanite.ProgrammableRaster` | 1 | プログラマブルラスタ（マテリアル評価あり） |
| `r.Nanite.MaxPixelsPerEdge` | 1.0 | LOD 目標辺長 |
| `r.Nanite.DicingRate` | 2.0 | テッセレーション micropolygon 目標 |
| `r.Nanite.MaxPatchesPerGroup` | 5 | テッセレーションパッチ/group |
| `r.Nanite.DepthBucketing` | 1 | 深度バケット振り分け（透明度フェード等） |

### 3.5 Tessellation Path（`NaniteRasterizer.usf:776-`）

`NANITE_TESSELLATION` 定義時は `PatchRasterize` 関数経由でランタイムテッセレーション → マイクロポリゴン dice → SW ラスタ。`r.Nanite.MaxPatchesPerGroup = 5` でグループあたりパッチ数上限。

---

## 4. 近似・省略の差分

| 項目 | HW ラスタ | Nanite SW ラスタ | 影響 |
|------|----------|----------------|------|
| Subpixel 精度 | 8.8（HW 仕様） | 8.8 (`NANITE_SUBPIXEL_SAMPLES=256`) | 同等 |
| Conservative Rasterization | HW サポート | 不要（小三角形のため） | — |
| Multisample | HW MSAA | 1 サンプル | TSR で代替 |
| Wireframe | HW | 1 ピクセル丸め | 細線 aliasing |
| 大三角形効率 | 圧倒的 | 走査面積に比例で重い | HW path に振分 |
| Quad Derivatives | HW 自動 | クラスタ境界で再計算 | マテリアル微分が境界で不連続 |

---

## 5. パラメータと CVar

§3.4 にまとめ済み。`r.Nanite.MinPixelsPerEdgeHW = 0` で全クラスタ HW にできる（デバッグ用）。

---

## 6. 代替手法との比較

| 手法 | 表現 | 小三角形効率 | 採用 |
|------|------|-----------|------|
| HW Rasterizer | 固定関数 | 低（quad 単位） | 全 GPU 標準 |
| **Nanite SW (compute)** | **エッジ関数 + InterlockedMax** | **高（1 三角形 1 スレッド）** | **Nanite 標準** |
| GPU Splatting | 点群 | 中 | Gaussian Splatting 系 |
| Voxel Rasterization | ボクセル | — | UE Voxel/Brick path（`AutoVoxel.usf` 等） |
| REYES（Pixar 古典） | マイクロポリゴン dice | 高 | オフラインレンダ |

### REYES vs Nanite

- REYES: dice → shade → sample（ハイエンド映像、CPU/オフライン）
- Nanite: cluster → cull → SW raster → shade（リアルタイム、GPU compute）
- 思想は近い（"shade ≃ pixel" 単位）が、Nanite はストリーミングと DAG LOD で実用化

---

## 7. 参考資料

- [S20] Karis / Stenson / Sherlock 2021 §3
- Olano & Greer 1997 "Triangle Scan Conversion using 2D Homogeneous Coordinates"
- 関連: [[nanite_cluster_culling]] / [[nanite_visibility_buffer]]
- Cook 1987 "The REYES Image Rendering Architecture"

---

## 8. 相談用フック

- **理解度チェック**:
  - SW ラスタが HW より速くなる三角形サイズ → 〜32 ピクセル辺以下
  - InterlockedMax(uint64) の役割 → 深度比較 + ID 書込を 1 アトミック
  - クラスタ内頂点キャッシュの利点 → 頂点重複計算回避、shared memory 活用
- **コード深掘り候補**:
  - `NaniteRasterizationCommon.ush` の `RasterizeTri_Adaptive` 適応分割ロジック
  - `SetupTriangle` の edge function 係数導出
  - `NaniteRasterBinning.usf` の SW/HW 振り分け
- **未読箇所**:
  - S20 §3.4 conservative coverage 議論
  - Tessellation path（`NaniteTessellation.ush` / `NaniteDice.ush`）
  - `Voxel/RasterizeBricks.usf` の Voxel ラスタ
- **次の派生**:
  - Tessellation アルゴリズム → [[nanite_tessellation]]（未着手）
  - Voxel ブリックラスタ → [[nanite_voxel_brick]]（未着手）
  - Visibility Buffer 出力統合 → [[nanite_visibility_buffer]]
