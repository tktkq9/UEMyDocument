# REF: Height Fog シェーダー

- グループ: a - Height Fog
- 詳細: [[detail_height_fog]]
- CPU リファレンス: [[ref_fog_rendering]]
- ソース: `Engine/Shaders/Private/HeightFogPixelShader.usf`  
          `Engine/Shaders/Private/HeightFogVertexShader.usf`

---

## ExponentialPixelMain（HeightFogPixelShader.usf:39）

### エントリポイント

```hlsl
void ExponentialPixelMain(
    float2 TexCoord    : TEXCOORD0,
    float3 ScreenVector: TEXCOORD1,
    float4 SVPos       : SV_POSITION,
    out float4 OutColor: SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Pixel Shader |
| **目的** | 指数霧または Volumetric Fog の結果を SceneColor に合成 |
| **ブレンド** | アディティブ（SrcBlend=SrcAlpha / DstBlend=InvSrcAlpha）|

#### パーミュテーション

| ディメンション | 説明 |
|-------------|------|
| `PERMUTATION_SUPPORT_LOCAL_FOG_VOLUME` | LocalFogVolume 対応 |
| `PERMUTATION_SAMPLE_FOG_ON_CLOUDS` | Volumetric Cloud の深度でフォグを計算 |

#### 入力バインド

| バインド名 | 型 | 説明 |
|-----------|------|------|
| `SceneDepthTexture` | `Texture2D` | SceneTextures 経由のシーン深度 |
| `FFogUniformParameters` | UB | 密度・高さ・色・Volumetric Fog 連携パラメータ |
| `IntegratedLightScattering` | `Texture3D` | Volumetric Fog 積分結果（使用時のみ）|
| `OcclusionTexture` | `Texture2D` | Sky AO（オプション）|
| `WaterDepthTexture` | `Texture2D` | 水面深度（`bUseWaterDepthTexture=1` 時）|

#### CPU バインド

```cpp
// FogRendering.cpp - RenderFog()
FHeightFogPS::FParameters* PassParameters = GraphBuilder.AllocParameters<FHeightFogPS::FParameters>();
PassParameters->Fog = CreateFogUniformBuffer(GraphBuilder, View);
PassParameters->SceneDepth = SceneTextures.SceneDepthTexture;
if (bShouldRenderVolumetricFog)
{
    PassParameters->IntegratedLightScattering = VolumetricFogIntegration.IntegratedLightScattering;
}

GraphBuilder.AddPass(
    RDG_EVENT_NAME("HeightFog"),
    PassParameters,
    ERDGPassFlags::Raster,
    [PassParameters](FRHICommandList& RHICmdList)
    {
        DrawFullscreenTriangle(RHICmdList, HeightFogVS, HeightFogPS, PassParameters);
    });
```

---

## HeightFogVertexShader（HeightFogVertexShader.usf）

```hlsl
void Main(
    in  float2 InPosition   : ATTRIBUTE0,
    out float4 OutPosition  : SV_POSITION,
    out float2 OutTexCoord  : TEXCOORD0,
    out float3 OutScreenVector : TEXCOORD1)
```

| 項目 | 内容 |
|-----|------|
| **目的** | フルスクリーントライアングルの TexCoord と ScreenVector（視線方向）を出力 |
| **OutScreenVector** | `View.ScreenToWorld` 変換でスクリーンピクセルの視線ベクトルを生成 |

---

## FFogUniformParameters 主要フィールド

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FFogUniformParameters,)
    SHADER_PARAMETER(FVector4f, ExponentialFogParameters)
    // .x = density × exp(-falloff × (camH - fogH))
    // .y = HeightFalloff
    // .z = MaxOpacity
    // .w = FogHeight

    SHADER_PARAMETER(float, ApplyVolumetricFog)      // 1.0 = Volumetric Fog 使用
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, IntegratedLightScattering)
    // Froxel 積分結果 Texture3D
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Fog` | 1 | Height Fog 有効 |
| `r.Fog.RenderFogOnClouds` | 0 | 雲の表面でフォグを計算 |
| `r.SupportLocalFogVolumes` | 0 | LocalFogVolume アクタ対応 |
