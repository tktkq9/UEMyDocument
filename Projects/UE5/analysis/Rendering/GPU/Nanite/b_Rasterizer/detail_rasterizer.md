# Nanite Rasterizer シェーダー詳細

- グループ: b - Rasterizer
- GPU 概要: [[01_nanite_gpu_overview]]
- CPU 詳細: [[a_nanite_cull_raster]]
- リファレンス: [[ref_rasterizer]]

---

## 概要

Nanite のラスタライズパイプライン。クラスターカリングの出力を入力として、  
**Raster Binning → SW Rasterizer / HW Rasterizer** の順に実行する。  
Pass 1（前フレーム HZB 後）と Pass 2（新 HZB 後）の2回このパイプラインが走る。

---

## パス構成

```
Raster Binning（NaniteRasterBinning.usf）
  ├─ RasterBinInit       … 全ビンをゼロクリア
  ├─ RasterBinBuild      … クラスターごとにマテリアルビンを決定・カウント
  ├─ RasterBinReserve    … ビンのバッファ範囲を確保（WaveInterlockedAdd）
  ├─ RasterBinFinalize   … Indirect Draw/Dispatch 引数を書き出し
  └─ RasterBinDepthBlock … Depth ブロック単位の分類（テッセレーション使用時）

SW Rasterizer（NaniteRasterizer.usf）
  └─ MicropolyRasterize  … 微小三角形を Compute でラスタライズ（VisBuffer64 書き込み）

HW Rasterizer（NaniteRasterizer.usf）
  ├─ HWRasterizeMS       … Mesh Shader パス（GPU がサポートする場合）
  └─ HWRasterizePS       … Vertex + Pixel Shader パス（フォールバック）
```

---

## Raster Binning のコアロジック

### ビン分類の概念

各クラスターはマテリアルに応じた **Raster Bin** に割り当てられる。  
同一マテリアルのクラスターを1回のディスパッチでまとめて処理することで、  
Material の定数バッファ切り替えコストを削減する。

```hlsl
// RasterBinBuild: クラスターごとに実行
[numthreads(64, 1, 1)]
void RasterBinBuild(uint RelativeClusterIndex : SV_DispatchThreadID, ...)
{
    FVisibleCluster VisibleCluster = GetVisibleCluster(VisibleClusterIndex, ...);
    FCluster Cluster = GetCluster(VisibleCluster.PageIndex, VisibleCluster.ClusterIndex);

    // クラスターの相対マテリアルインデックス → グローバルRasterBin番号
    const uint RasterBin = GetRemappedRasterBinFromIndex(
        GetRelativeMaterialIndex(Cluster, TriIndex),
        InstanceData.PrimitiveId,
        RegularMaterialRasterBinCount,
        RenderFlags, ...
    );

    // SW / HW の判定: クラスタ面積が閾値以下なら SW
    ExportRasterBin(RasterBin, BinMaterialFlags, ClusterIndex,
                    RangeStart, RangeEnd, BatchCount, ..., bSoftware, ...);
}

// RasterBinReserve: Wave単位でバッファ範囲を確保
[numthreads(64, 1, 1)]
void RasterBinReserve(uint RasterBinIndex : SV_DispatchThreadID)
{
    const uint RangeStart = AllocateRasterBinRange(RasterBinCapacity);
    SetRasterBinOffset(RasterBinIndex, RangeStart);
    // Indirect Draw/Dispatch 引数も書き込み
}
```

### Raster Bin パイプラインの流れ

```
RasterBinInit      → 全ビンの Count/Offset をゼロ初期化
RasterBinBuild     → クラスター → ビン割り当て・カウントインクリメント
RasterBinReserve   → カウントを元に各ビンのバッファ領域を連続確保
RasterBinBuild(2)  → 実際のクラスターデータをオフセット位置に書き込み
RasterBinFinalize  → Indirect Args 最終化（Draw/Dispatch 引数）
```

---

## SW Rasterizer（MicropolyRasterize）のコアロジック

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void MicropolyRasterize(
    uint DispatchThreadID : SV_DispatchThreadID,
    uint GroupID          : SV_GroupID,
    uint GroupIndex       : SV_GroupIndex
)
{
    // NANITE_VOXELS, PATCHES, 通常クラスターで処理を切り替え
    ClusterRasterize(GroupID, GroupIndex);
}
```

`ClusterRasterize()` の内部処理：
1. VisibleCluster を取得し、変換行列でクリップ空間に投影
2. 各三角形を遍歴し、ピクセルカバレッジを計算
3. **VisBuffer64** に depth（R24）+ VisibleClusterIndex（+TriIndex）を Atomically 書き込み
4. `InterlockedMax()` で最前面のピクセルのみ残す

### THREADGROUP_SIZE の分岐

| 条件 | サイズ | 理由 |
|-----|--------|------|
| Voxel / VertReuseBatch | 32 | 1 Wave = 1 クラスター |
| 通常クラスター | 64 | 2 Wave = 1 クラスター（キャッシュ効率）|

---

## HW Rasterizer のコアロジック

大きな三角形は HW パイプライン（Mesh Shader または VS+PS）でラスタライズする。  
Depth-only パスは `NANITE_HW_RASTER_INTERPOLATE_DEPTH` が有効になり、精度が向上する。

```hlsl
// Mesh Shader パス（GPU サポート時）
void HWRasterizeMS(uint GroupThreadID, uint3 GroupID, ...)
{
    // 1 GroupThread = 1 頂点 or 1 三角形の処理
    // SetMeshOutputCounts(VertexCount, TriangleCount) で出力をセット
}

// Pixel Shader パス
void HWRasterizePS(VSOut In, ...)
{
    // 通常の PS として実行
    // SV_Depth で depth を上書き（NANITE_HW_RASTER_INTERPOLATE_DEPTH 時）
    // VisBuffer64 への AtomicMax 書き込み
}
```

---

## VisBuffer64 のフォーマット

| ビット | 内容 |
|--------|------|
| 63-40 | Depth（R24UNorm, Reverse-Z）|
| 39-7 | VisibleClusterIndex（33bit）|
| 6-0 | TriIndex（7bit, クラスター内三角形番号）|

SW/HW どちらのラスタライザも `VisBuffer64` に `InterlockedMax` で書き込む。  
Reverse-Z のため、より前面（大きい浮動小数）のピクセルが勝ち残る。

---

## 入出力

### 入力

| リソース | 内容 |
|---------|------|
| `VisibleClustersSWHW` | カリング後の可視クラスターリスト |
| `ClusterPageData` | クラスター頂点・三角形データ |
| `InstanceSceneData` | インスタンス変換行列 |
| `NaniteViews` | カメラ / ライト / VSM 各ビュー |
| `RasterBinMeta` | ビンごとのマテリアルフラグ・容量 |

### 出力

| リソース | 内容 |
|---------|------|
| `RWVisBuffer64` | メイン Visibility Buffer（depth + clusterID + triIdx）|
| `RWDbgBuffer64/32` | デバッグ統計バッファ（オプション）|
| `OutRasterBinArgsSWHW` | Raster Bin ごとの Indirect Args |

---

## CPU 呼び出しの流れ

```
Nanite::RasterizePass()                        // NaniteCullRaster.cpp
  │
  ├─ RasterBinInit CS
  ├─ RasterBinBuild CS（Pass 1）
  ├─ RasterBinReserve CS
  ├─ RasterBinBuild CS（Pass 2: 実際のデータ書き込み）
  ├─ RasterBinFinalize CS
  │
  ├─ MicropolyRasterize（SW — Indirect Dispatch per Bin）
  └─ HWRasterizeMS / HWRasterizePS（HW — Indirect Draw per Bin）
```
