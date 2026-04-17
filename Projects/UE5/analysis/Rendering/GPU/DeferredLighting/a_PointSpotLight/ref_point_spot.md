# REF: Point / Spot Light Deferred Shading

- グループ: a - PointSpotLight
- 詳細: [[detail_point_spot]]
- CPU リファレンス: [[ref_deferred_lighting]]
- ソース: `Engine/Shaders/Private/ClusteredDeferredShadingPixelShader.usf`

---

## ClusteredShadingPixelShader（:309, :373, :434）

### エントリポイント（Substrate Tile Type 別）

```hlsl
// SUBSTRATE_TILETYPE = 0: FastPath
void ClusteredShadingPixelShader(float4 SvPosition : SV_POSITION, out float4 OutColor : SV_Target0)

// SUBSTRATE_TILETYPE = 1: SinglePath
void ClusteredShadingPixelShader(float4 SvPosition : SV_POSITION, out float4 OutColor : SV_Target0)

// SUBSTRATE_TILETYPE = 2〜3: ComplexPath（フル Substrate BSDF ループ）
void ClusteredShadingPixelShader(float4 SvPosition : SV_POSITION, out float4 OutColor : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Pixel Shader |
| **目的** | LightGrid のクラスター内全ローカルライトのライティングを GBuffer に適用 |
| **ブレンド** | Additive（SceneColor += ライティング結果）|

#### パーミュテーション

| マクロ | 説明 |
|-------|------|
| `SUBSTRATE_TILETYPE` | 0=FastPath / 1=SinglePath / 2,3=ComplexPath |
| `USE_TRANSMISSION` | Subsurface Transmission 対応 |
| `USE_IES_PROFILE` | IES ライトプロファイル |
| `SUBSTRATE_ENABLED` | Substrate マテリアル対応 |

#### CPU バインド

```cpp
// ClusteredDeferredShading.cpp
FClusteredDeferredShadingParameters* PassParameters = ...;
PassParameters->LightGridData = LightGridUniformBuffer;
PassParameters->LightDataBuffer = GPUScene.LightDataBuffer;
PassParameters->SceneTextures = SceneTextureUniformBuffer;

GraphBuilder.AddPass(
    RDG_EVENT_NAME("ClusteredDeferredShading"),
    PassParameters,
    ERDGPassFlags::Raster,
    [PassParameters](FRHICommandList& RHICmdList)
    {
        DrawFullscreenTriangle(RHICmdList, VS, PS_TileType, PassParameters);
    });
```

---

## DeferredLightPixelShaders.usf（ディレクショナルライト）

DeferredLightPixelShaders.usf は通常の関数定義を含まず、  
`DeferredLightingCommon.ush` の `GetDynamicLighting()` を呼び出す PP include 形式。  
実体は `PIXEL_SHADER_WRITES_DEPTH_BUFFER_STENCIL_BIT` 等のマクロで生成される。

```hlsl
// DeferredLightingCommon.ush より
FDeferredLightingSplit GetDynamicLighting(
    float3 TranslatedWorldPosition,
    float3 CameraVector,
    FGBufferData GBuffer,
    float AmbientOcclusion,
    uint ShadingModelID,
    FDeferredLightData LightData,
    ...)
// → Diffuse / Specular / Transmission を返す
```

---

## LightGridCommon.ush 主要関数

```hlsl
// クラスターインデックスの計算
uint ComputeLightGridCellIndex(uint2 PixelPos, float SceneDepth)

// クラスター内のライト数・インデックスを取得
uint GetNumLocalLights(uint GridIndex)
uint GetLocalLightIndex(uint GridIndex, uint i)
```
