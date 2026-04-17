# REF: Sky Light / Reflection Environment

- グループ: d - SkyLight
- 詳細: [[detail_sky_light]]
- CPU リファレンス: [[ref_deferred_lighting]]
- ソース: `Engine/Shaders/Private/ReflectionEnvironmentPixelShader.usf`  
          `Engine/Shaders/Private/ReflectionEnvironmentShaders.usf`

---

## ReflectionEnvironmentPixelShader.usf

DeferredLightPixelShaders.usf と同様、`#include "PixelShaderOutputCommon.ush"` 経由でエントリポイントを生成。  
`FPixelShaderInOut_MainPS()` 内で `GetImageBasedReflectionLighting()` を呼び出す。

### GetImageBasedReflectionLighting（ReflectionEnvironmentShared.ush）

```hlsl
float3 GetImageBasedReflectionLighting(
    FGBufferData GBuffer,
    float3 CameraVector,
    float3 TranslatedWorldPosition,
    float AmbientOcclusion,
    float IndirectIrradiance)
// → Specular IBL（ReflectionCapture × GF テクスチャ）を返す
```

---

## ComputeReflectionCaptureMipFromRoughness（ReflectionEnvironmentShared.ush）

```hlsl
float ComputeReflectionCaptureMipFromRoughness(float Roughness)
{
    // Roughness → ミップレベル（0=鏡面 / MaxMip=完全拡散）
    float LevelFrom1x1 = REFLECTION_CAPTURE_ROUGHEST_MIP - REFLECTION_CAPTURE_ROUGHNESS_MIPSCALE * log2(Roughness);
    return REFLECTION_CAPTURE_ROUGHEST_MIP - LevelFrom1x1;
}
```

---

## ReflectionCapture パラメータ（ReflectionEnvironmentShared.ush）

```hlsl
// 各 ReflectionCapture の情報
struct FReflectionCapture
{
    float4 PositionAndRadius;          // xyz=位置, w=影響半径
    float4 CaptureProperties;          // x=亮度, y=距離, zw=パラメータ
    float4 BoxTransform[4];            // OBB 変換（Box Capture 用）
    float4 BoxScales;                  // OBB スケール
};
```

---

## Pre-Integrated GF テクスチャ

```hlsl
// PreIntegratedGFTexture[NoV, Roughness]
// .r = ScaleGF （Fresnel 項の近似スケール）
// .g = BiasGF  （Fresnel 項の近似バイアス）
// F_approx = F0 * ScaleGF + BiasGF
// → BRDF の数値積分を事前テクスチャ化して高速化
float2 GF = PreIntegratedGF.SampleLevel(Sampler, float2(NoV, Roughness), 0);
float3 SpecularColor = GBuffer.F0 * GF.r + GF.g;
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.ReflectionEnvironment` | 1 | Reflection Environment 有効 |
| `r.ReflectionEnvironment.ScreenSpaceReflections` | 1 | SSR 有効 |
| `r.ReflectionCaptureResolution` | 128 | ReflectionCapture キューブマップ解像度 |
| `r.SkyLight.RealTimeCapture` | 0 | リアルタイム Sky Light キャプチャ |
