# ref: BasePass 共通パラメータ

- 対象: `BasePassRendering.h` / `BasePassCommon.ush`
- Details: [[c_basepass_material]]

---

## FOpaqueBasePassUniformParameters（不透明用 UniformBuffer）

```cpp
// BasePassRendering.h:89
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FOpaqueBasePassUniformParameters,)
    // 共通パラメータ（Forward / Reflection / Fog / SkyLight 等）
    SHADER_PARAMETER_STRUCT(FSharedBasePassUniformParameters, Shared)
    // Substrate GBuffer エンコードパラメータ
    SHADER_PARAMETER_STRUCT(FSubstrateBasePassUniformParameters, Substrate)
    // Forward Shading 用 Screen Space Shadow Mask
    SHADER_PARAMETER(int32, UseForwardScreenSpaceShadowMask)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ForwardScreenSpaceShadowMaskTexture)
    // SSAO テクスチャ
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, IndirectOcclusionTexture)
    // 解決済み深度（MSAA 対応）
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ResolvedSceneDepthTexture)
    // DBuffer デカール
    SHADER_PARAMETER_STRUCT_INCLUDE(FDBufferParameters, DBuffer)
    // Pre-integrated GF（IBL スペキュラー用 LUT）
    SHADER_PARAMETER_TEXTURE(Texture2D, PreIntegratedGFTexture)
    SHADER_PARAMETER_SAMPLER(SamplerState, PreIntegratedGFSampler)
    // 深度フォーマット
    SHADER_PARAMETER(int32, Is24BitUnormDepthStencil)
    // Eye Adaptation（自動露出）
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, EyeAdaptationBuffer)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

---

## FSharedBasePassUniformParameters（共通 UniformBuffer）

```cpp
// BasePassRendering.h:78
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FSharedBasePassUniformParameters,)
    // Forward Shading 用ライトデータ
    SHADER_PARAMETER_STRUCT(FForwardLightUniformParameters, Forward)
    // Reflection Capture（IBL キューブマップ）
    SHADER_PARAMETER_STRUCT(FReflectionUniformParameters, Reflection)
    // Planar Reflection（水面等）
    SHADER_PARAMETER_STRUCT(FPlanarReflectionUniformParameters, PlanarReflection)
    // Height Fog
    SHADER_PARAMETER_STRUCT(FFogUniformParameters, Fog)
    SHADER_PARAMETER_STRUCT(FFogUniformParameters, FogISR)  // Instance Stereo Rendering用
    // Local Fog Volume
    SHADER_PARAMETER_STRUCT(FLocalFogVolumeUniformParameters, LFV)
    // Light Function Atlas
    SHADER_PARAMETER_STRUCT(LightFunctionAtlas::FLightFunctionAtlasGlobalParameters, LightFunctionAtlas)
    // スカイライト使用フラグ
    SHADER_PARAMETER(uint32, UseBasePassSkylight)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

---

## FTranslucentBasePassUniformParameters（半透明用 UniformBuffer）

```cpp
// BasePassRendering.h:106
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FTranslucentBasePassUniformParameters,)
    SHADER_PARAMETER_STRUCT(FSharedBasePassUniformParameters, Shared)
    SHADER_PARAMETER_STRUCT(FSceneTextureUniformParameters, SceneTextures) // GBuffer 参照
    SHADER_PARAMETER_STRUCT(FSubstrateForwardPassUniformParameters, Substrate)
    SHADER_PARAMETER_STRUCT(FOITBasePassUniformParameters, OIT)  // Order-Independent Translucency
    // SSR（Screen Space Reflection）
    SHADER_PARAMETER_STRUCT_INCLUDE(FHZBParameters, HZBParameters)
    SHADER_PARAMETER(FVector4f, PrevScreenPositionScaleBias)
    SHADER_PARAMETER(int32, SSRQuality)
    SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture2D, PrevSceneColor)  // 前フレームカラー
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

---

## FForwardLightData（Forward Shading 用ライストデータ）

```cpp
// BasePassRendering.h:59
class FForwardLightData
{
    // 1ライスト分のパックデータ
    FVector4f LightPositionAndInvRadius;
    FVector4f LightColorAndIdAndFalloffExponentAndRayEndBias;
    FVector4f LightDirectionAndSceneInfoExtraDataPacked;
    FVector4f SpotAnglesAndSourceRadiusPacked;
    FVector4f LightTangentAndIESDataAndSpecularScale;
    FVector4f RectDataAndVirtualShadowMapIdOrPrevLocalLightIndex;
};
```

---

> [!note]- IMPLEMENT_GLOBAL_SHADER_PARAMETER_STRUCT マクロの役割
> `SHADER_PARAMETER_STRUCT` でネストされた構造体はシェーダー内では `%StructName%.%FieldName%` の形式でアクセスできる。
> `FOpaqueBasePassUniformParameters` は HLSL 側では `BasePass` という名前でバインドされる（`IMPLEMENT_GLOBAL_SHADER_PARAMETER_STRUCT` の第2引数）。

> [!note]- DBuffer との統合
> `FDBufferParameters` を `SHADER_PARAMETER_STRUCT_INCLUDE` で埋め込むことで、
> DBuffer デカールが存在する場合にシェーダー内で自動的に適用される。
> `r.DBuffer=0` の場合は DBuffer テクスチャが Dummy テクスチャに差し替えられる。
