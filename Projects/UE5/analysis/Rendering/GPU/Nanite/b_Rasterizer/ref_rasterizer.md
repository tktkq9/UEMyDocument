# REF: Nanite Rasterizer シェーダー

- グループ: b - Rasterizer
- 詳細: [[detail_rasterizer]]
- CPU リファレンス: [[ref_nanite_cull_raster]]
- ソース: `Engine/Shaders/Private/Nanite/NaniteRasterBinning.usf`
          `Engine/Shaders/Private/Nanite/NaniteRasterizer.usf`

---

## NaniteRasterBinning.usf

### InitVisiblePatchesArgs

```hlsl
[numthreads(1, 1, 1)]
void InitVisiblePatchesArgs()
```

| 目的 | テッセレーションパッチの Indirect Dispatch 引数を初期化 |

### RasterBinInit

```hlsl
[numthreads(64, 1, 1)]
void RasterBinInit(uint RasterBinIndex : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 全ラスタービンの Count・Offset フィールドをゼロ初期化 |
| **CPU 関数** | `Nanite::RasterizePass()` 内の先頭ステップ |

### RasterBinBuild

```hlsl
[numthreads(64, 1, 1)]
void RasterBinBuild(
    uint RelativeClusterIndex : SV_DispatchThreadID,
    uint GroupThreadIndex     : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 可視クラスター→マテリアルビン割り当て。SW/HW フラグを付けてクラスターをビンリストに追加 |
| **入力** | `VisibleClustersSWHW`, `ClusterPageData`, `RasterBinMeta`, `RegularMaterialRasterBinCount` |
| **出力** | `RWRasterBinData`（ビン別クラスターリスト）, `RWRasterBinMeta`（カウント）|
| **CPU 関数** | `Nanite::RasterizePass()` |

### RasterBinReserve

```hlsl
[numthreads(64, 1, 1)]
void RasterBinReserve(uint RasterBinIndex : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 各ビンの必要バッファ量を `WaveInterlockedAdd` でアトミックに確保し、オフセットを決定 |
| **出力** | `OutRangeAllocator`（累積オフセット）, `OutRasterBinArgsSWHW`（Indirect Args 初期化）|

### RasterBinFinalize

```hlsl
[numthreads(64, 1, 1)]
void RasterBinFinalize(uint RasterBinIndex : SV_DispatchThreadID)
```

| 目的 | 各ビンの最終クラスター数から Indirect Dispatch/Draw 引数を確定して書き込む |
| 出力 | `OutRasterBinArgsSWHW`（各ビンのスレッドグループ数 or 頂点数）|

### RasterBinDepthBlock

```hlsl
[numthreads(64, 1, 1)]
void RasterBinDepthBlock(
    uint DepthBlockIndex : SV_GroupID,
    uint ThreadIndex     : SV_GroupIndex
)
```

| 目的 | テッセレーション時の Depth Block（パッチ深度バケット）をビン別に分類 |

---

## NaniteRasterizer.usf

### MicropolyRasterize（SW ラスタライザ）

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void MicropolyRasterize(
    uint DispatchThreadID : SV_DispatchThreadID,
    uint GroupID          : SV_GroupID,
    uint GroupIndex       : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Compute シェーダーで微小三角形（Sub-pixel 〜数 px）を直接ラスタライズし VisBuffer64 に書き込む |
| **THREADGROUP_SIZE** | 通常=64, Voxel/VertReuseBatch=32 |
| **内部関数** | `ClusterRasterize()` / `PatchRasterize()` / `ClusterTraceBricks()`（Voxel）|
| **出力** | `RWVisBuffer64`（InterlockedMax で深度+クラスター情報を書き込み）|
| **CPU 関数** | `Nanite::RasterizePass()` — Indirect Dispatch per Bin |

### HWRasterizeMS（HW Mesh Shader ラスタライザ）

```hlsl
MESH_SHADER_TRIANGLE_ATTRIBUTES(NANITE_MESH_SHADER_TG_SIZE)
void HWRasterizeMS(
    uint GroupThreadID : SV_GroupThreadID,
    uint3 GroupID      : SV_GroupID,
    MESH_SHADER_VERTEX_EXPORT(VSOut, 256),
    MESH_SHADER_TRIANGLE_EXPORT(128),
    MESH_SHADER_PRIMITIVE_EXPORT(PrimitiveAttributesPacked, 128)
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Mesh Shader パスで大三角形を HW ラスタライズ（VS+PS より高効率） |
| **条件** | `NANITE_MESH_SHADER=1`（GPU サポート時）|
| **出力** | HW パイプラインで `HWRasterizePS` に渡す |

### HWRasterizePS（HW Pixel Shader ラスタライザ）

```hlsl
void HWRasterizePS(
    VSOut In
    [, PrimitiveAttributesPacked PrimitivePacked]  // Mesh Shader 時
    [, bool bFrontFace : SV_IsFrontFace]            // TwoSided 時
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | HW ラスタライズのピクセルシェーダー — VisBuffer64 への InterlockedMax 書き込み |
| **NANITE_HW_RASTER_INTERPOLATE_DEPTH** | 深度を手動補間（Depth-Only 時の高精度モード）|
| **出力** | `RWVisBuffer64`（UAV AtomicMax）/ または SV_Depth（通常 PS パス）|
| **CPU 関数** | `Nanite::RasterizePass()` — Indirect Draw per Bin |

---

## パーミュテーション概要

| マクロ | 値 | 説明 |
|--------|-----|------|
| `COMPUTESHADER` | 0/1 | SW（CS）vs HW（VS+PS）|
| `NANITE_MESH_SHADER` | 0/1 | Mesh Shader パスを使用 |
| `DEPTH_ONLY` | 0/1 | Shadow/Depth パス（GBuffer 出力なし）|
| `NANITE_TWO_SIDED` | 0/1 | 両面マテリアル |
| `NANITE_TESSELLATION` | 0/1 | マイクロポリゴンテッセレーション |
| `NANITE_VOXELS` | 0/1 | ボクセルモード |
| `VIRTUAL_TEXTURE_TARGET` | 0/1 | VSM（Virtual Shadow Map）向け |

---

> [!note]- SW と HW ラスタライザの使い分け基準
> `RasterBinBuild` で各クラスターが SW/HW どちらのビンに入るかを決定する。  
> 判断基準はクラスターの **投影面積（ピクセル数換算）**: 数ピクセル以下の微小クラスターは SW（Compute）に、  
> それ以上は HW（Mesh Shader/VS+PS）に振り分ける。  
> SW の方がスレッドグループの管理コストが低く、微小三角形では HW の固定機能ロジックよりも効率的。

> [!note]- VisBuffer64 の AtomicMax による深度テスト
> 通常の HW 深度バッファは固定機能で「小さい（手前の）深度を残す」が、  
> VisBuffer64 は Reverse-Z のため「大きい値が手前」となる。  
> InterlockedMax によって複数スレッドが同一ピクセルに書き込んでも正しく最前面のピクセルが勝つ。  
> ただし Depth と ClusterInfo が1つの uint64 にパックされているため、分割更新は発生しない。

> [!note]- RasterBin の2フェーズ（Init→Build→Reserve→Build→Finalize）
> `RasterBinBuild` は2回呼ばれる: 1回目はカウントのみ加算し、`RasterBinReserve` でオフセットを確保した後、  
> 2回目でそのオフセット位置にクラスターを実際に書き込む。  
> これは GPU 上でのダイナミック配列アロケーションのイディオム（Prefix Sum 相当）であり、  
> 1パスで実現するには `RasterBinReserve` に相当するアトミック演算が必要になる。
