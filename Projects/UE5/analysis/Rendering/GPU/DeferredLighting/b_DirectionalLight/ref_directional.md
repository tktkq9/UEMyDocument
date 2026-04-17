# REF: Directional Light Deferred Shading

- グループ: b - DirectionalLight
- 詳細: [[detail_directional]]
- CPU リファレンス: [[ref_deferred_lighting]]
- ソース: `Engine/Shaders/Private/DeferredLightPixelShaders.usf`  
          `Engine/Shaders/Private/DeferredLightingCommon.ush`

---

## DeferredLightPixelShaders.usf（LIGHT_TYPE_DIRECTIONAL）

DeferredLightPixelShaders.usf は直接エントリポイントを定義せず、  
`#include "PixelShaderOutputCommon.ush"` 経由で `FPixelShaderInOut_MainPS()` を生成する。  
実体は `DeferredLightingCommon.ush#GetDynamicLighting()` を呼び出す形。

### ライトタイプ定義

| マクロ | 値 | 説明 |
|-------|------|------|
| `LIGHT_TYPE_DIRECTIONAL` | 0 | ディレクショナルライト |
| `LIGHT_TYPE_POINT` | 1 | ポイントライト |
| `LIGHT_TYPE_SPOT` | 2 | スポットライト |
| `LIGHT_TYPE_RECT` | 3 | 矩形ライト（Area Light）|

---

## FDeferredLightData（DeferredLightingCommon.ush）

```hlsl
struct FDeferredLightData
{
    float3 TranslatedWorldPosition;  // ライト位置（Directional は無視）
    float  InvRadius;                // 1 / InfluenceRadius（Directional は 0）
    float3 Color;                    // ライト色 × 強度
    float  FalloffExponent;          // 減衰指数

    float3 Direction;                // ライト方向（Directional のみ使用）
    float  SpecularScale;

    float2 SpotAngles;               // CosOuter, CosInner（Spot のみ）
    float  SourceRadius;             // 光源半径（Area Light）
    float  SoftSourceRadius;

    bool bInverseSquared;            // 逆二乗減衰フラグ
    bool bRadialLight;               // 放射状ライト（Point/Spot）
    bool bSpotLight;                 // スポットライト
    bool bRectLight;                 // 矩形ライト
};
```

---

## GetDynamicLighting（DeferredLightingCommon.ush）

```hlsl
FDeferredLightingSplit GetDynamicLighting(
    float3 TranslatedWorldPosition,
    float3 CameraVector,
    FGBufferData GBuffer,
    float AmbientOcclusion,
    uint ShadingModelID,
    FDeferredLightData LightData,
    float4 LightAttenuation,    // シャドウマスク（R=静的, G=動的）
    float2 Random,              // ランダムサンプル（PCSS用）
    uint2 SVPos,
    FTranslucencyLightingParameters TranslucencyLightingParameters)
```

#### 戻り値

```hlsl
struct FDeferredLightingSplit
{
    float4 DiffuseLighting;    // .rgb = Diffuse ライティング
    float4 SpecularLighting;   // .rgb = Specular ライティング
};
```

---

## Shadow Quality パーミュテーション

| `SHADOW_QUALITY` | PCF サンプル数 | 用途 |
|-----------------|------------|------|
| 1 | 1 サンプル | 最低品質 |
| 2 | 4 サンプル | 低品質 |
| 3 | 16 サンプル | 中品質 |
| 4 | 16 サンプル（irregular）| 高品質 |
| 5 | 32 サンプル | 最高品質（PCSS）|

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Shadow.Quality` | 5 | シャドウ PCF サンプル数 |
| `r.Shadow.CSM.MaxCascades` | 4 | Cascaded Shadow Map 最大分割数 |
| `r.CapsuleShadows` | 1 | カプセルシャドウ（キャラクター用）|
