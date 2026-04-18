# GPU a: LightSampling — 確率的サンプリング + 可視性テスト

- シェーダー: `MegaLights/MegaLightsSampling.usf`, `MegaLightsRayTracing.usf`, `MegaLightsHardwareRayTracing.usf`
- CPU 対応: [[b_megalights_sampling]]
- 上位: [[01_megalights_gpu_overview]]

---

## 概要

各ピクセルに対して ClusteredLightGrid から候補ライトを収集し、  
BlueNoise + 重要度サンプリングで少数（デフォルト 4 個）のライトを確率的に選択する。  
選択後は各サンプルに対してシャドウレイを生成し、可視性（0/1）を記録する。

---

## GenerateLightSamplesCS（MegaLightsSampling.usf: line 304）

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void GenerateLightSamplesCS(
    uint3 GroupId : SV_GroupID,
    uint3 GroupThreadId : SV_GroupThreadID,
    uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // 1. DownsampledTileData から処理タイル座標を取得
    // 2. LoadMaterial() で GBuffer の深度・法線・マテリアル情報を読み込み
    // 3. ComputeLightGridCellIndex() で ClusteredLightGrid のセルを特定
    // 4. 各候補ライトの EstimatedIrradiance（強度 × NdotL × 距離減衰）を計算
    // 5. GUIDE_BY_HISTORY > 0 時: 前フレーム VisibleLightHash で PDF を補正
    // 6. BlueNoise テクスチャから乱数を生成
    // 7. CDF バイナリサーチで NUM_SAMPLES_PER_PIXEL_1D 個のライトを選択
    // 8. 選択ライットの Index, Weight, Direction を LightSamples に書き込み
    // 9. シャドウレイ（Origin, Direction, TMax）を LightSampleRays に書き込み
}
```

**パーミュテーション:**

| マクロ | 値 | 説明 |
|--------|-----|------|
| `TILE_TYPE` | `TILE_MODE_*` (0–12) | タイル種別（Simple / Single / Complex / Special × Rect / Textured）|
| `NUM_SAMPLES_PER_PIXEL_1D` | 1, 2, 4, 16 | ピクセルあたりサンプル数（1D）|
| `GUIDE_BY_HISTORY` | 0/1/2 | History ガイドの強度 |
| `DEBUG_MODE` | 0/1 | デバッグ ShaderPrint 有効 |

---

## サンプリング戦略

```
EstimatedIrradiance[i] = LightIntensity[i]
                        × saturate(dot(Normal, LightDir[i]))
                        × DistanceAttenuation(Distance[i], InvRadius[i])

PDF[i] = EstimatedIrradiance[i] / Σ(EstimatedIrradiance)

[GUIDE_BY_HISTORY]
  VisibleLightHashHistory から前フレームの可視情報を取得
  Visibility = 1 だったライト → PDF を引き上げ
  Visibility = 0 だったライト → PDF を引き下げ
  → 再正規化

[サンプル選択]
  u = BlueNoise[PixelX, PixelY, FrameIndex % SliceCount]
  LightIndex[s] = CDF バイナリサーチ(u)
  SampleWeight[s] = 1.0 / (PDF[LightIndex[s]] × NumSamplesPerPixel)
```

---

## ClearLightSamplesCS（MegaLightsSampling.usf: line 721）

毎フレーム LightSamples バッファをクリアする補助 CS。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ClearLightSamplesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

---

## 可視性テスト（MegaLightsRayTracing.usf）

### CompactLightSampleTracesCS（line 83）

VisibleLightHash から追跡が必要なサンプルをコンパクトリストに収集する。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void CompactLightSampleTracesCS(...)
// → CompactedTraceTexelIndirectArgs に Indirect Dispatch 引数を書き込み
```

### ScreenSpaceRayTraceLightSamplesCS（line 226）

スクリーン空間でのレイトレース（コスト低）。  
HZB を使って画面内にヒットするか判定する。

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void ScreenSpaceRayTraceLightSamplesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

### SoftwareRayTraceLightSamplesCS（line 351）

Global Distance Field を使ったソフトウェアレイトレース。  
HW RT 非対応環境でのフォールバック。

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void SoftwareRayTraceLightSamplesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

### InitCompactedTraceTexelIndirectArgsCS（line 28）

Indirect Dispatch 引数を初期化する補助 CS。

---

## HW Ray Tracing（MegaLightsHardwareRayTracing.usf）

`EnabledRT` モード時、RayGen シェーダーで各サンプルに対してハードウェアレイを発射。  
`RayQuery` でシャドウの遮蔽を判定し LightSamples.Visibility を 0.0 / 1.0 で記録する。

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.MegaLights.NumSamplesPerPixel` | 4 | ピクセルあたりサンプル数 |
| `r.MegaLights.DownsampleMode` | 2 | 0=なし / 1=チェッカー / 2=半解像度 |
| `r.MegaLights.MinSampleWeight` | 0.001 | 最小サンプル重み |
| `r.MegaLights.GuideByHistory` | 2 | History ガイド強度 |
| `r.MegaLights.HardwareRayTracing` | 1 | HW RT 使用（0=ソフトウェア RT）|
