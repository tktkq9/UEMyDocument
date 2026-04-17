# ref: Clustered Deferred Shading クラス群

- 対象: `ClusteredDeferredShadingPass.cpp`
- Details: [[b_clustered_tiled]]

---

## FClusteredShadingPS（クラスタードシェーディング PS）

```cpp
// ClusteredDeferredShadingPass.cpp:102
class FClusteredShadingPS : public FGlobalShader
{
    DECLARE_GLOBAL_SHADER(FClusteredShadingPS);
    SHADER_USE_PARAMETER_STRUCT(FClusteredShadingPS, FGlobalShader)

    // パーミュテーション
    class FVisualizeLightCullingDim : SHADER_PERMUTATION_BOOL("VISUALIZE_LIGHT_CULLING");
    class FHairStrandsLighting       : SHADER_PERMUTATION_BOOL("USE_HAIR_LIGHTING");
    class FLightFunctionAtlasDim     : SHADER_PERMUTATION_BOOL("USE_LIGHT_FUNCTION_ATLAS");
    class FRectLight                 : SHADER_PERMUTATION_BOOL("USE_RECT_LIGHT");
    class FAnistropicMaterials       : SHADER_PERMUTATION_BOOL("SUPPORTS_ANISOTROPIC_MATERIALS");
    // Substrate タイルタイプ（Fast/Single/Complex）もパーミュテーションに含む
    using FPermutationDomain = TShaderPermutationDomain<
        FVisualizeLightCullingDim,
        FHairStrandsLighting,
        Substrate::FSubstrateTileType,
        FLightFunctionAtlasDim,
        FRectLight,
        FAnistropicMaterials>;

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        // Forward Light Grid（ライストインデックスリスト）
        SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FForwardLightUniformParameters, ForwardLightStruct)
        SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
        SHADER_PARAMETER_STRUCT_INCLUDE(ShaderPrint::FShaderParameters, ShaderPrintUniformBuffer)
        SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FHairStrandsViewUniformParameters, HairStrands)
        SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSubstrateGlobalUniformParameters, Substrate)
        // GBuffer テクスチャ群
        SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneTextureUniformParameters, SceneTextures)
        // Light Function Atlas（バッチ化ライストファンクション）
        SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FLightFunctionAtlasGlobalParameters, LightFunctionAtlas)
        SHADER_PARAMETER_STRUCT_INCLUDE(FSceneLightingChannelParameters, LightingChannelParameters)
        // VSM Shadow Mask（MegaLights 等の前処理結果）
        SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ShadowMaskBits)
        SHADER_PARAMETER_RDG_TEXTURE(Texture2D<uint>, LightingChannelsTexture)
        // Virtual Shadow Map サンプリングパラメータ
        SHADER_PARAMETER_STRUCT_INCLUDE(FVirtualShadowMapSamplingParameters, VirtualShadowMapSamplingParameters)
        SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, HairTransmittanceBuffer)
        RENDER_TARGET_BINDING_SLOTS()
    END_SHADER_PARAMETER_STRUCT()
};

// USFパス: /Engine/Private/ClusteredDeferredShadingPixelShader.usf
IMPLEMENT_GLOBAL_SHADER(FClusteredShadingPS, "...", "ClusteredShadingPixelShader", SF_Pixel);
```

---

## FClusteredShadingVS（頂点シェーダー）

```cpp
// ClusteredDeferredShadingPass.cpp:74
class FClusteredShadingVS : public FGlobalShader
{
    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
        SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FHairStrandsViewUniformParameters, HairStrands)
        SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneTextureUniformParameters, SceneTextures)
    END_SHADER_PARAMETER_STRUCT()
};
// USFパス: /Engine/Private/ClusteredDeferredShadingVertexShader.usf
```

---

## EClusterPassInputType（入力タイプ）

```cpp
// ClusteredDeferredShadingPass.cpp:176
enum class EClusterPassInputType : uint8
{
    GBuffer,      // 通常の不透明 GBuffer
    Substrate,    // Substrate マテリアルシステム
    HairStrands   // Hair Strands 専用パス
};
```

---

## InternalAddClusteredDeferredShadingPass（内部実装）

```cpp
// ClusteredDeferredShadingPass.cpp:183
static void InternalAddClusteredDeferredShadingPass(
    FRDGBuilder& GraphBuilder,
    int32 ViewIndex,
    FViewInfo& View,
    const FMinimalSceneTextures& SceneTextures,
    const FSortedLightSetSceneInfo& SortedLightsSet,
    EClusterPassInputType InputType,
    ESubstrateTileType TileType,
    FRDGTextureRef LightingChannelsTexture,
    FRDGTextureRef ShadowMaskBits,
    FVirtualShadowMapArray& VirtualShadowMapArray,
    FRDGBufferSRVRef HairTransmittanceBuffer,
    FSubstrateSceneData* SubstrateSceneData)
{
    // SortedLightsSet.ClusteredSupportedEnd > 0 であることを確認
    // ForwardLightUniformBuffer からライストグリッドを参照
    // SceneTextures から GBuffer を読み取り BRDF 評価
    // ShadowMaskBits = null の場合は Zero ダミーテクスチャで代替
}
```

---

## 使用条件チェック

```cpp
// ClusteredDeferredShadingPass.cpp:57
bool FDeferredShadingSceneRenderer::ShouldUseClusteredDeferredShading(EShaderPlatform InPlatform) const
{
    return CVarClusteredDeferredShadingEnableForProject.GetValueOnAnyThread() > 0
        && GUseClusteredDeferredShading != 0
        && Scene->GetFeatureLevel() >= ERHIFeatureLevel::SM6
        && DoesPlatformSupportVirtualShadowMaps(InPlatform);
}
// r.UseClusteredDeferredShading_ToBeRemoved=1（デフォルト 0）が必要
// SM6 以上かつ VSM 対応プラットフォームのみ有効
```

---

> [!note]- Clustered Deferred は将来削除予定
> `r.UseClusteredDeferredShading` の CVarDeprecated コメントに
> "NOTE: The clustered deferred shading implementation will be removed in a future release due to low utility and thus use."
> とあり、今後は MegaLights が多灯処理の主役となる。

> [!note]- Light Grid との統合
> `FForwardLightUniformParameters` 内の `ForwardLightGrid` テクスチャが
> 3D クラスターグリッド（FroxelGrid）のライストインデックスリストを持つ。
> シェーダーはスクリーン UV + 深度から Froxel インデックスを計算し、
> そのインデックスに格納されたライストリストを順に評価する。

> [!note]- Substrate タイルとのパーミュテーション
> `Substrate::FSubstrateTileType` がパーミュテーションに入るため、
> Fast / Single / Complex の3タイルで別シェーダーがコンパイルされる。
> タイルごとに Stencil 描画エリアが限定されるため、複雑なマテリアルのみ
> 重いシェーダーが走り帯域・ALU を節約できる。
