# REF: Lumen Scene Direct Lighting シェーダー

- グループ: b - Scene Lighting
- 詳細: [[detail_direct_lighting]]
- CPU リファレンス: [[ref_lumen_scene_lighting]]
- ソース: `Engine/Shaders/Private/Lumen/LumenSceneDirectLighting.usf`  
          `Engine/Shaders/Private/Lumen/LumenSceneDirectLightingCulling.usf`  
          `Engine/Shaders/Private/Lumen/LumenSceneDirectLightingShadowMask.usf`  
          `Engine/Shaders/Private/Lumen/LumenSceneDirectLightingSoftwareRayTracing.usf`  
          `Engine/Shaders/Private/Lumen/LumenSceneDirectLightingHardwareRayTracing.usf`  
          `Engine/Shaders/Private/Lumen/LumenSceneLighting.usf`

---

## LumenSceneDirectLighting.usf

### LumenCardBatchDirectLightingCS

```hlsl
[numthreads(CARD_TILE_SIZE, CARD_TILE_SIZE, 1)]   // CARD_TILE_SIZE=8 → 64スレッド/グループ
void LumenCardBatchDirectLightingCS(
    uint3 GroupId        : SV_GroupID,
    uint3 GroupThreadId  : SV_GroupThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 各 Card タイルにバッチされたライト群の直接照明を計算 |
| **入力** | `CardTiles`（カードタイル配列）, `LightTilesPerCardTile`（ライトタイルリスト）, `AlbedoAtlas`, `NormalAtlas` |
| **出力** | `RWDirectLightingAtlas`（Surface Cache 直接光アトラス）|
| **CPU 関数** | `RenderBatchedLights()` in `LumenSceneDirectLighting.cpp` |

#### パーミュテーション

| クラス | 説明 |
|-------|------|
| `bHasMultipleViews` | マルチビュー対応（VR 等）|
| `WAVE_OP_WAVE_SIZE` | Wave サイズ最適化（SM6 WAVESIZE attribute）|

---

## LumenSceneLighting.usf

### CombineLumenSceneLightingCS

```hlsl
[numthreads(CARD_TILE_SIZE, CARD_TILE_SIZE, 1)]
void CombineLumenSceneLightingCS(
    uint3 GroupId       : SV_GroupID,
    uint3 GroupThreadId : SV_GroupThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Direct + Indirect + Emissive を合成して FinalLightingAtlas を生成 |
| **計算式** | `FinalLighting = Albedo × (Direct + Indirect) + Emissive` |
| **入力** | `AlbedoAtlas`, `EmissiveAtlas`, `DirectLightingAtlas`, `IndirectLightingAtlas` |
| **出力** | `RWFinalLightingAtlas`（最終照明アトラス。Screen Probe トレースのヒット輝度として使用）|
| **CPU 関数** | `CombineLumenSceneLighting()` in `LumenSceneLighting.cpp` |

### ClearCardUpdateContextCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void ClearCardUpdateContextCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 目的 | 毎フレーム冒頭で更新コンテキストバッファをゼロクリア |
|-----|------|

### BuildPageUpdatePriorityHistogramCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void BuildPageUpdatePriorityHistogramCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 目的 | CardPage の更新優先度（距離・変化量等）をヒストグラムに集計 |
|-----|------|
| **更新コンテキスト** | `CARD_UPDATE_CONTEXT_DIRECT_LIGHTING` と `CARD_UPDATE_CONTEXT_INDIRECT_LIGHTING` の2種 |

### SelectMaxUpdateBucketCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void SelectMaxUpdateBucketCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 目的 | ヒストグラムから「今フレームで更新するバケット上限」を決定（予算制御）|
|-----|------|

### BuildCardsUpdateListCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void BuildCardsUpdateListCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 目的 | 選択されたバケット内の Card を更新リストに追加 |
|-----|------|
| **出力** | `RWCardPageIndexAllocator`, `RWCardPageIndexData`（更新対象 CardPage インデックス配列）|

### SetCardPageIndexIndirectArgsCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void SetCardPageIndexIndirectArgsCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 目的 | Indirect Dispatch / DrawIndirect の引数テクスチャを設定 |
|-----|------|

### RasterizeToCardsVS

```hlsl
void RasterizeToCardsVS(
    uint CardIndex   : SV_VertexID,
    out float4 OutPosition : SV_POSITION,
    out float2 OutCardUV  : TEXCOORD0)
```

| 目的 | Card の UV 矩形をスクリーン三角形にマップ（ラスタライズ版ライティング用）|
|-----|------|

### ClearLumenCardsPS

```hlsl
void ClearLumenCardsPS(out float4 OutTarget0 : SV_Target0)
```

| 目的 | カード領域を 0 にクリア（Direct / Indirect 再計算前）|
|-----|------|

---

## LumenSceneDirectLightingCulling.usf（カリング）

主要 CS（詳細はソース参照）：

| エントリポイント | 役割 |
|---------------|------|
| `BuildLightTilesCS()` | カードに影響するライトのタイルリストを構築 |
| `SplatLightTileToCardTilesCS()` | ライトタイルを CardTile に逆マッピング |
| `InitializeCardTileCS()` | 各 CardTile の初期化 |

---

## LumenSceneDirectLightingShadowMask.usf（シャドウマスク）

| エントリポイント | 役割 |
|---------------|------|
| `InitShadowMaskCS()` | シャドウマスクバッファの初期化 |
| `ComposeShadowMasksCS()` | 複数シャドウソースのマスクを合成 |

---

## LumenSceneDirectLightingHardwareRayTracing.usf（HW RT 版シャドウ）

```hlsl
// DXR レイ生成シェーダー（r.Lumen.HardwareRayTracing=1 時）
RAY_TRACING_ENTRY_RAYGEN(LumenSceneDirectLightingRayGenRGS)
```

| 項目 | 内容 |
|-----|------|
| **目的** | DXR でシャドウレイをトレースして Surface Cache ライティングの影を計算 |
| **CPU 関数** | `DispatchLumenDirectLightingHardwareRayTracingIndirect()` |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Lumen.SurfaceCache.DirectLighting.MaxShadowDistance` | 10000 | シャドウレイの最大距離 |
| `r.Lumen.SurfaceCache.DirectLighting.CardUpdateFrequency` | 2 | 更新フレーム間隔（1=毎フレーム）|
| `r.Lumen.HardwareRayTracing` | 0 | HW RT シャドウの使用 |
| `r.Lumen.SurfaceCache.DirectLighting.StochasticLighting` | 0 | 確率的ライティング（多数ライト向け）|

---

> [!note]- DirectLightingAtlas と FinalLightingAtlas の違い
> `DirectLightingAtlas` は「直接光のみの Irradiance」（W/m²）を格納する中間バッファ。  
> `FinalLightingAtlas` は `Albedo × (Direct + Indirect) + Emissive` の最終輝度（nit）を格納する。  
> Screen Probe や Radiance Cache がサーフェスをヒットしたとき参照するのは `FinalLightingAtlas` で、Albedo は含まれている。

> [!note]- Card 更新バジェット制御の仕組み
> 毎フレーム全 Card を更新するとコストが高い。`BuildPageUpdatePriorityHistogramCS()` → `SelectMaxUpdateBucketCS()` でフレームごとの更新バジェットを超えないよう「バケット制限」をかける。  
> 優先度は距離・移動速度・ライト変化量などで決まり、遠距離・静止オブジェクトの更新は間引かれる（`r.Lumen.SurfaceCache.DirectLighting.CardUpdateFrequency` で制御）。

> [!note]- Wave サイズ最適化とカードタイル
> `LumenCardBatchDirectLightingCS` は `CARD_TILE_SIZE=8`（8×8=64 スレッド）で動作する。  
> SM6 対応 GPU では `WAVESIZE(WAVE_OP_WAVE_SIZE)` 属性で Wave サイズを 32 または 64 に固定し、同じタイル内の異なるライトの Shadow 判定を Wave-level 操作で並列化している。
