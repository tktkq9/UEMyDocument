# ref: FSkyLightSceneProxy / FSkyLightParameters / ConvolveSpecularSkyLight

- 対象ファイル: `ReflectionEnvironmentCapture.h/.cpp` / `SkyAtmosphereRendering.h`
- 概要: [[18_skyatmosphere_overview]]

---

## FSkyLightSceneProxy の主要フィールド

```cpp
// SkyLight.h（Engine 側 → レンダースレッドに FSkyLightSceneProxy として渡される）
class FSkyLightSceneProxy
{
    // ─── キャプチャ方式 ──────────────────────────────────────────
    bool bRealTimeCaptureEnabled;       // リアルタイムキャプチャ（毎フレーム更新）
    float BlendFraction;                // リアルタイム↔静的のブレンド比率

    // ─── ライティングプロパティ ──────────────────────────────────
    FLinearColor LightColor;            // Sky Light の色と輝度
    float Intensity;                    // 強度
    bool bCastShadows;
    bool bCastRaytracedShadow;
    bool bCloudAmbientOcclusion;        // Volumetric Cloud の AO 適用

    // ─── キャプチャ済みテクスチャ ────────────────────────────────
    FTexture* ProcessedSkyTexture;      // フィルタ済みキューブマップ（事前焼き込み版）
    FTextureCubeRHIRef ProcessedSkyTextureRef;
    FSHVectorRGB3 IrradianceEnvironmentMap; // L2 SH 係数（拡散間接光）
    float AverageBrightness;            // 平均輝度（DFAO 参照強度に使用）

    // ─── SkyAtmosphere 連携 ─────────────────────────────────────
    FSkyAtmosphereRenderSceneInfo* SkyAtmosphereInfo;
    bool bHasStaticSkyLight;            // 静的 Sky Light か
    bool bWantsStaticShadowing;
};
```

---

## FSkyLightParameters（ReflectionEnvironmentCapture.h）

```cpp
// DeferredShading.usf などでバインドされる Sky Light パラメータ
BEGIN_SHADER_PARAMETER_STRUCT(FSkyLightParameters, )
    SHADER_PARAMETER_TEXTURE(TextureCube, SkyLightCubemap)       // フィルタ済みキューブマップ
    SHADER_PARAMETER_TEXTURE(TextureCube, SkyLightBlendDestinationCubemap) // ブレンド先
    SHADER_PARAMETER_SAMPLER(SamplerState, SkyLightCubemapSampler)
    SHADER_PARAMETER(float, SkyLightCubemapBrightness)            // 輝度スケール
    SHADER_PARAMETER(float, SkyLightBlendFraction)                // ブレンド比率
    SHADER_PARAMETER(int32, SkyLightCubemapNumMips)               // Mip 数
    SHADER_PARAMETER(FVector4f, SkyLightColor)                    // 色 × 強度
    SHADER_PARAMETER_ARRAY(FVector4f, SkyIrradianceEnvironmentMap, [7])
    // → L2 SH 係数（7 要素にパック済み）
END_SHADER_PARAMETER_STRUCT()
```

---

## FReflectionUniformParameters（ReflectionEnvironmentCapture.h）

```cpp
// ReflectionCapture / Sky Light 共通のパラメータ
BEGIN_UNIFORM_BUFFER_STRUCT(FReflectionUniformParameters, )
    SHADER_PARAMETER(FVector4f, SkyLightParameters)     // Sky Light 輝度等
    SHADER_PARAMETER_TEXTURE(TextureCubeArray, ReflectionCubemap)
    // → ReflectionCapture のキューブマップ配列
    SHADER_PARAMETER_SAMPLER(SamplerState, ReflectionCubemapSampler)
    SHADER_PARAMETER(float, ReflectionCubemapMaxMip)
    // ... 他フィールド ...
END_UNIFORM_BUFFER_STRUCT()
```

---

## ConvolveSpecularSkyLight()（ReflectionEnvironmentCapture.cpp）

```
ConvolveSpecularSkyLight(
    FRHICommandListImmediate& RHICmdList,
    FScene* Scene,
    FTextureCubeRHIRef SrcCubemap,
    FTextureCubeRHIRef& OutFilteredTexture,
    int32 CubemapSize)

処理内容:
  for each Mip in [0, MipCount):
    Roughness = Mip / (MipCount - 1)    // Mip0 = Roughness=0（鏡面）, MipN = Roughness=1（拡散）

    for each CubeFace in 6:
      → GGX 分布の Importance Sampling でフィルタ（GGX_HorizonFade を使用）
      → SampleCount は Roughness が高いほど増やす
      → GPU CS: FilterCS.usf → OutFilteredTexture[Mip][Face] に書き込み

出力テクスチャ:
  フォーマット: PF_FloatRGBA（または PF_BC6H 事前圧縮版）
  解像度: 128×128×6 × MipCount（デフォルト）
  → DeferredLighting で IBL（Image-Based Lighting）として使用
```

---

## SkyLight の DeferredLighting 利用

```
[スペキュラ IBL（ReflectionEnvironment.usf）]
  float Roughness = GBufferB.a
  int MipLevel = log2(SkyLightCubemapNumMips) × Roughness
  float3 R = reflect(-V, N)
  float3 Specular = SkyLightCubemap.SampleLevel(R, MipLevel) × SkyLightCubemapBrightness

  // エネルギー保存: PreIntegratedGFTexture で GGX 補正
  float2 AB = PreIntegratedGFTexture.SampleLevel(float2(NoV, Roughness), 0)
  SpecularColor = F0 × AB.x + AB.y

[拡散 IBL（拡散間接光）]
  // L2 SH からの拡散照明
  float3 Irradiance = SHEval(SkyIrradianceEnvironmentMap, N)
  float3 Diffuse = Irradiance × DiffuseColor / π

[遮蔽]
  float SkyOcclusion = ScreenSpaceAO or DFAO
  Diffuse × SkyOcclusion
  Specular × pow(SkyOcclusion, SpecularOcclusionExponent)
```

---

## 関連リファレンス

- [[ref_sky_atmosphere]] — `FAtmosphereUniformShaderParameters` / `FSkyAtmosphereRenderContext`
- [[b_sky_light]] — SkyLight キャプチャ・Convolve フロー
