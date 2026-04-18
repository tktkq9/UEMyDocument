# GPU c: Composite — タイル分類 + ボリューム照明

- シェーダー: `MegaLights/MegaLights.usf`, `MegaLightsVolumeSampling.usf`, `MegaLightsVolumeRayTracing.usf`, `MegaLightsVolumeShading.usf`
- CPU 対応: [[a_megalights_pipeline]], [[d_megalights_resolve]]
- 上位: [[01_megalights_gpu_overview]]

---

## 概要

このグループは MegaLights パイプラインの**前処理（タイル分類）**と  
**後処理（半透明ボリューム照明）**を担当する。

---

## MegaLightsTileClassificationBuildListsCS（MegaLights.usf: line 70）

スクリーン空間タイルを Substrate マテリアルタイプ別に分類し、  
後続の CS（GenerateLightSamplesCS / ShadeLightSamplesCS）用のタイルリストを構築する。

```hlsl
[numthreads(THREADGROUP_SIZE * THREADGROUP_SIZE, 1, 1)]
void MegaLightsTileClassificationBuildListsCS(
    uint2 GroupId : SV_GroupID,
    uint GroupThreadId : SV_GroupThreadID)
{
    // 1. Z-Order カーブで LocalTileCoord を計算
    // 2. [DOWNSAMPLE_FACTOR_X != 1]
    //    ダウンサンプル領域のタイルビットマスクを MegaLightsTileBitmask から収集
    // 3. GetTileModeFromBitmask() で TILE_MODE を決定
    //    - MEGALIGHTS_TILE_BITMASK_COMPLEX_SPECIAL → COMPLEX_SPECIAL
    //    - MEGALIGHTS_TILE_BITMASK_COMPLEX → COMPLEX
    //    - MEGALIGHTS_TILE_BITMASK_SINGLE → SINGLE
    //    - MEGALIGHTS_TILE_BITMASK_SIMPLE → SIMPLE
    // 4. WaveActiveCountBits() でタイル数を波数単位でカウント
    // 5. InterlockedAdd(RWTileAllocator[TileTypeIndex], ...) でオフセット取得
    // 6. RWTileData[Offset + TileTypeIndex × Stride] にタイル座標を書き込み
}
```

**入出力:**

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `MegaLightsTileBitmask` | `Texture2D<uint>` | Substrate タイルビットマスク（入力）|
| `RWTileAllocator` | `RWBuffer<uint>` | タイルタイプ別カウンタ |
| `RWTileData` | `RWBuffer<uint>` | タイル座標（TILE_TYPE × TileDataStride）|
| `RWDownsampledTileAllocator` | `RWBuffer<uint>` | ダウンサンプルタイルカウンタ |
| `RWDownsampledTileData` | `RWBuffer<uint>` | ダウンサンプルタイル座標 |

---

## InitTileIndirectArgsCS（MegaLights.usf: line 136）

タイルリストから Indirect Dispatch 引数を初期化する補助 CS。

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void InitTileIndirectArgsCS(uint3 DispatchThreadId : SV_DispatchThreadID)
// TileAllocator[TileTypeIndex] → RWTileIndirectArgs
```

---

## HairTransmittanceCS（MegaLights.usf: line 188）

Hair Strands 向けのトランスミッタンス計算 CS。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void HairTransmittanceCS(...)
```

---

## ボリューム照明（半透明用 3D テクスチャ）

### VolumeGenerateLightSamplesCS（MegaLightsVolumeSampling.usf: line 161）

フロクセル（Froxel）単位でライットを確率的サンプリングする 3D CS。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, THREADGROUP_SIZE)]
void VolumeGenerateLightSamplesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // 1. Froxel インデックス → ワールド座標に変換
    // 2. ClusteredLightGrid から Froxel 内の候補ライットを取得
    // 3. PDF 計算 + BlueNoise サンプリングでライットを選択
    // 4. シャドウレイを VolumeLightSampleRays に書き込み
}
```

### VolumeCompactLightSampleTracesCS（MegaLightsVolumeRayTracing.usf: line 36）

ボリューム向けコンパクト化 CS（Froxel 単位）。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, THREADGROUP_SIZE)]
void VolumeCompactLightSampleTracesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

### VolumeSoftwareRayTraceLightSamplesCS（MegaLightsVolumeRayTracing.usf: line 140）

GDF（Global Distance Field）を使ったボリューム向けソフトウェアレイトレース。

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void VolumeSoftwareRayTraceLightSamplesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

### VolumeShadeLightSamplesCS（MegaLightsVolumeShading.usf: line 99）

フロクセル単位でライティングを評価し、半透明ボリュームテクスチャに書き込む。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, THREADGROUP_SIZE)]
void VolumeShadeLightSamplesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // 1. LoadPackedLightSamples() でフロクセルのサンプルを取得
    // 2. ForwardLightData からライット情報を取得
    // 3. Henyey-Greenstein 位相関数 (VolumePhaseG) でボリューム散乱を評価
    // 4. Ambient（SH L0）→ TranslucencyAmbient[Cascade]
    //    Directional（SH L1）→ TranslucencyDirectional[Cascade]
    //    に書き込み
}
```

---

## ボリュームテクスチャ仕様

| テクスチャ | 型 | 内容 |
|--------|-----|------|
| `TranslucencyAmbient[TVC_MAX]` | `RWTexture3D<float4>` | アンビエント照明（SH L0）|
| `TranslucencyDirectional[TVC_MAX]` | `RWTexture3D<float4>` | 方向性照明（SH L1）|

- `TVC_MAX` = TranslucencyVolumeCascadeCount（カスケード数）
- 半透明レンダリング時: `TranslucentBasePassUniformParameters.MegaLightsVolume` 経由で参照

---

## ボリューム照明フロー

```
VolumeGenerateLightSamplesCS（各 Froxel でサンプリング）
  ↓
[EnabledVSM] VSM 参照 / [EnabledRT] MegaLightsVolumeHardwareRayTracing
  ↓
VolumeShadeLightSamplesCS（Henyey-Greenstein 評価 → SH 書き込み）
  ↓
TranslucencyVolumeShading.usf でサンプリング
  → 半透明マテリアルの Emissive / IndirectLighting に加算
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.MegaLights.Volume` | 1 | TranslucencyVolume の有効/無効 |
| `r.MegaLights` | 0 | MegaLights 全体有効（0=無効/1=VSM/2=RT）|
