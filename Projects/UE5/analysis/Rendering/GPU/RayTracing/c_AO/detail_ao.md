# GPU c: AO — Ray Traced Ambient Occlusion + SkyLight

- シェーダー: `RayTracing/RayTracingAmbientOcclusionRGS.usf`, `RayTracingSkyLightRGS.usf`, `GenerateSkyLightVisibilityRaysCS.usf`, `CompositeAmbientOcclusionPS.usf`, `CompositeSkyLightPS.usf`
- CPU 対応: [[b_rt_shadow_ao]]
- 上位: [[01_raytracing_gpu_overview]]

---

## 概要

- **RT AO**: 各ピクセルからコサインサンプリングで半球レイを発射し、近距離の遮蔽を計算する。
- **RT SkyLight**: 天空方向へのレイで空の遮蔽マスクを生成し、Sky Light 輝度に掛け合わせる。

---

## AmbientOcclusionRGS（RayTracingAmbientOcclusionRGS.usf: line 85）

Substrate GBuffer から法線・深度を読み込み、ヘミスフェアランダムレイで遮蔽を積算する。

```hlsl
RAY_TRACING_ENTRY_RAYGEN(AmbientOcclusionRGS)
{
    const uint2 PixelCoord = DispatchRaysIndex().xy + View.ViewRectMin.xy;

    // 1. SceneDepthTexture からワールド位置を再構築
    // 2. [SUBSTRATE_GBUFFER_FORMAT==1]
    //    SubstrateTopLayerData から法線・BSDF を取得
    //    [従来 GBuffer]
    //    GetGBufferData() から法線を取得
    // 3. GenerateCosineNormalRay() でランダムレイ生成
    //    Direction = コサインサンプリング（半球）
    //    TMax = MaxRayDistance
    // 4. for (SampleIndex in [0, SamplesPerPixel)):
    //    TraceRay(TLAS, RAY_TRACING_MASK_OPAQUE, RayDesc)
    //    IsMiss() → Visibility += RayWeight × 1.0
    //    IsHit()  → Visibility += RayWeight × (1.0 - IntensityLocal)
    // 5. RWAmbientOcclusionMaskUAV[PixelCoord] = Visibility / SamplesPerPixel
    // 6. RWAmbientOcclusionHitDistanceUAV = 平均ヒット距離（デノイザー向け）
}
```

**パラメータ:**

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `TLAS` | `RaytracingAccelerationStructure` | シーン TLAS |
| `SamplesPerPixel` | `uint` | ピクセルあたりサンプル数 |
| `MaxRayDistance` | `float` | AO の最大距離（cm 単位）|
| `Intensity` | `float` | AO 強度スケール |
| `MaxNormalBias` | `float` | 法線バイアス上限 |
| `RWAmbientOcclusionMaskUAV` | `RWTexture2D<float>` | 出力: 遮蔽マスク（0=完全遮蔽/1=非遮蔽）|
| `RWAmbientOcclusionHitDistanceUAV` | `RWTexture2D<float>` | 出力: ヒット距離（デノイズ用）|

**コサインサンプリング:**

```hlsl
// GenerateCosineNormalRay() の概要
float4 Direction_Tangent = CosineSampleHemisphereConcentric(RandSample);
// CONFIG_SHOOT_WITH_GEOMETRIC_NORMAL=1 の場合、幾何法線でサンプリング
float3 Direction_World = TangentToWorld(Direction_Tangent.xyz, GeometricNormal);
RayPDF = Direction_Tangent.w;  // cos θ / π
```

---

## SkyLightRGS（RayTracingSkyLightRGS.usf: line 32）

スカイライト方向へのレイで天空遮蔽（Sky Occlusion）を計算する。  
レイがヒットした場合は遮蔽とし、Miss の場合は Sky Light 輝度を評価する。

```hlsl
RAY_TRACING_ENTRY_RAYGEN(SkyLightRGS)
{
    uint2 PixelCoord = GetPixelCoord(DispatchRaysIndex().xy, UpscaleFactor);

    // 1. SceneDepthTexture → TranslatedWorldPosition 再構築
    // 2. [SUBSTRATE_GBUFFER_FORMAT==1] Substrate から法線・BSDF 取得
    // 3. PathTracingRandomSequence でランダムシーケンス初期化
    // 4. RayTracingSkyLightEvaluation.ush: SampleSkyLightVisibility()
    //    → 天空方向をサンプリング + TraceRay() で遮蔽判定
    //    → SkyLightSample.Radiance × Visibility を積算
    // 5. RWSkyOcclusionMaskUAV[PixelCoord] = 積算結果
    // 6. RWSkyOcclusionRayDistanceUAV = ヒット距離
}
```

**パラメータ:**

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `TLAS` | `RaytracingAccelerationStructure` | シーン TLAS |
| `UpscaleFactor` | `uint` | アップスケール係数 |
| `RWSkyOcclusionMaskUAV` | `RWTexture2D<float4>` | 出力: Sky 遮蔽マスク |
| `RWSkyOcclusionRayDistanceUAV` | `RWTexture2D<float2>` | 出力: ヒット距離 |

---

## GenerateSkyLightVisibilityRaysCS（GenerateSkyLightVisibilityRaysCS.usf: line 18）

SkyLight 可視性レイの方向を事前生成する補助 CS。  
MipTree ベースの重要度サンプリングで天空方向を選択する。

```hlsl
[numthreads(TILE_SIZE, TILE_SIZE, 1)]
void MainCS(uint3 DispatchThreadId : SV_DispatchThreadID)
// → SkyLightVisibilityRaysBuffer に (Direction, PDF) を書き込み
```

---

## 合成パス

### CompositeAmbientOcclusionPS.usf

デノイズ済みの AO マスクをスクリーンスペースに合成する PS。  
AO マスク × SceneColor を乗算して間接照明を暗くする。

### CompositeSkyLightPS.usf

デノイズ済みの SkyLight 遮蔽マスクを SceneColor に合成する PS。

---

## AO/SkyLight パイプライン

```
[RT AO]
AmbientOcclusionRGS
  → AmbientOcclusionMask（ノイジー）
  → GScreenSpaceDenoiser->DenoiseAmbientOcclusion()
  → CompositeAmbientOcclusionPS
  → SceneColor に AO 適用

[RT SkyLight]
GenerateSkyLightVisibilityRaysCS（方向生成）
  → SkyLightRGS（遮蔽テスト）
  → デノイズ
  → CompositeSkyLightPS
  → SceneColor にスカイライト適用
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RayTracing.AmbientOcclusion` | -1 | -1=自動 / 0=無効 / 1=有効 |
| `r.RayTracing.AmbientOcclusion.SamplesPerPixel` | 1 | AO サンプル数 |
| `r.RayTracing.AmbientOcclusion.MaxRayDistance` | 10000 | 最大 AO 距離（cm）|
| `r.RayTracing.SkyLight` | 1 | RT SkyLight 有効 |
| `r.RayTracing.SkyLight.SamplesPerPixel` | 4 | SkyLight サンプル数 |
