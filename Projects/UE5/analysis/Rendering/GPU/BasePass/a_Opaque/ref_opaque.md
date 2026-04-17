# REF: Base Pass Opaque シェーダー

- グループ: a - Opaque
- 詳細: [[detail_opaque]]
- CPU リファレンス: [[ref_basepass_gbuffer]]
- ソース: `Engine/Shaders/Private/BasePassVertexShader.usf`  
          `Engine/Shaders/Private/BasePassPixelShader.usf`

---

## BasePassVertexShader.usf#Main（:32）

### エントリポイント

```hlsl
void Main(
    FVertexFactoryInput Input,
    out FBasePassVSOutput Output
#if USE_GLOBAL_CLIP_PLANE
    , out float OutGlobalClipPlaneDistance : SV_ClipDistance
#endif
    )
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Vertex Shader |
| **目的** | WPO 適用 + 接線空間/ライトマップ UV を補間値として出力 |

#### FBasePassVSOutput 主要フィールド（BasePassVertexCommon.ush）

```hlsl
struct FBasePassVSToPS
{
    float4 Position : SV_POSITION;
    FVertexFactoryInterpolantsVSToPS FactoryInterpolants;
    FBasePassInterpolantsVSToPS BasePassInterpolants;
    // BasePassInterpolants には以下が含まれる:
    //   float4 PixelPosition (ワールド位置)
    //   float3 SkyAtmosphereLightDirection (SkyAtmosphere 有効時)
    //   float3 AmbientLightingVector (頂点フォグ・半透明時)
    //   float4 FogColor (半透明フォグ)
};
```

#### パーミュテーション

| マクロ | 説明 |
|-------|------|
| `TRANSLUCENCY_PERVERTEX_FORWARD_SHADING` | 半透明メッシュの頂点フォワードライティング |
| `NEEDS_BASEPASS_VERTEX_FOGGING` | 頂点フォグ（半透明用）|
| `LOCAL_FOG_VOLUME_ON_TRANSLUCENT` | LocalFogVolume 対応 |

---

## BasePassPixelShader.usf（内部 FPixelShaderInOut_MainPS）

| 項目 | 内容 |
|-----|------|
| **タイプ** | Pixel Shader |
| **目的** | マテリアルグラフ評価 → GBuffer 書き込み |

#### 主要ユニフォームバッファ

| バッファ | 説明 |
|---------|------|
| `OpaqueBasePass` | 不透明 BasePass 用 UB（SceneTextures / PreIntegratedGF 等）|
| `TranslucentBasePass` | 半透明 BasePass 用 UB |
| `View` | ビュー共通パラメータ |

#### GBuffer エンコード関数（ShaderPrint.ush / ShadingModelMaterials.ush）

```hlsl
// ShadingModel に応じて GBuffer フィールドを設定
void SetGBufferForShadingModel(
    inout FGBufferData GBuffer,
    const FMaterialPixelParameters MaterialParameters,
    const float Opacity,
    ...)
{
    GBuffer.Metallic  = Metallic;
    GBuffer.Specular  = Specular;
    GBuffer.Roughness = Roughness;
    GBuffer.WorldNormal = Normal;
    GBuffer.BaseColor = BaseColor;
    GBuffer.ShadingModelID = SHADINGMODELID_DEFAULT_LIT;
    // CustomData = Subsurface / Anisotropy / Clearcoat 等
}
```

---

## GBuffer ShadingModel ID

| 値 | 名前 | 説明 |
|----|------|------|
| 0 | `SHADINGMODELID_UNLIT` | アンリット |
| 1 | `SHADINGMODELID_DEFAULT_LIT` | 標準 PBR |
| 2 | `SHADINGMODELID_SUBSURFACE` | サブサーフェス散乱 |
| 3 | `SHADINGMODELID_PREINTEGRATED_SKIN` | 皮膚 |
| 4 | `SHADINGMODELID_CLEAR_COAT` | クリアコート |
| 5 | `SHADINGMODELID_SUBSURFACE_PROFILE` | サブサーフェスプロファイル |
| 6 | `SHADINGMODELID_TWOSIDED_FOLIAGE` | 両面葉 |
| 7 | `SHADINGMODELID_HAIR` | ヘア |
| 8 | `SHADINGMODELID_CLOTH` | クロス |
| 9 | `SHADINGMODELID_EYE` | 眼球 |
| 10 | `SHADINGMODELID_SINGLELAYERWATER` | 水 |
| 11 | `SHADINGMODELID_THIN_TRANSLUCENT` | 薄い半透明 |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.GBufferFormat` | 1 | GBuffer フォーマット（0=Low/1=Default/3=High）|
| `r.BasePassOutputsVelocity` | 0 | BasePass で Velocity 出力 |
| `r.EarlyZPassMovable` | 0 | 動的メッシュも EarlyZ に含める |
