# REF: Light Functions シェーダー

- グループ: c - LightFunctions
- 詳細: [[detail_light_functions]]
- CPU リファレンス: [[ref_deferred_lighting]]
- ソース: `Engine/Shaders/Private/LightFunctionPixelShader.usf`  
          `Engine/Shaders/Private/LightFunctionAtlas/LightFunctionAtlasRender.usf`

---

## LightFunctionPixelShader.usf#Main

```hlsl
void Main(
    float2 TexCoord     : TEXCOORD0,
    float4 SvPosition   : SV_POSITION,
    out float4 OutColor : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Pixel Shader |
| **目的** | ライト空間 UV からマテリアルを評価して Light Function 値（色・強度）を出力 |
| **出力** | Emissive チャンネルを Light Function として使用（RGB = ライト変調色）|

#### パーミュテーション

| マクロ | 説明 |
|-------|------|
| `LIGHT_TYPE` | DIRECTIONAL / POINT / SPOT / RECT |
| `INVERSE_SQUARED_FALLOFF` | 逆二乗減衰フラグ |

---

## LightFunctionAtlasRender.usf

```hlsl
// Atlas の各タイルに Light Function をレンダリング
void LightFunctionAtlasRenderPS(
    float2 TexCoord : TEXCOORD0,
    out float4 OutColor : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 全 Light Function マテリアルを 1 枚のアトラスに一括レンダリング |
| **Atlas サイズ** | `r.LightFunctionAtlas.Resolution`（デフォルト 512）|

---

## LightFunctionAtlasCommon.ush 主要関数

```hlsl
// アトラス UV 計算
float2 GetLightFunctionAtlasUV(FDeferredLightData LightData, float3 WorldPosition)
{
    // ワールド座標 → ライト空間 → アトラス UV に変換
    float3 LightSpacePos = mul(float4(WorldPosition, 1), LightData.WorldToLight);
    float2 LightFunctionUV = LightSpacePos.xy * 0.5 + 0.5;
    return LightFunctionUV * LightData.AtlasTileScale + LightData.AtlasTileOffset;
}
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.LightFunctionAtlas` | 1 | Light Function Atlas 有効（UE5.3+）|
| `r.LightFunctionAtlas.Resolution` | 512 | Atlas テクスチャ解像度 |
| `r.LightFunctionAtlas.MaxLights` | 256 | Atlas に収められる最大ライト数 |
