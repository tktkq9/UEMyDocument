# Directional Light Deferred Shading シェーダー詳細

- グループ: b - DirectionalLight
- GPU 概要: [[01_deferred_lighting_gpu_overview]]
- CPU 詳細: [[a_directional_light]]
- リファレンス: [[ref_directional]]

---

## 概要

ディレクショナルライト（太陽光）は全画面フルスクリーンパスで GBuffer を読み取り、  
PCF / PCSS シャドウと大気散乱（Sky Atmosphere）を統合してライティングを計算する。  
同じシェーダーファイル（DeferredLightPixelShaders.usf）を使いつつ、  
`LIGHT_TYPE` マクロでポイント/スポット/ディレクショナルの各タイプを切り替える。

---

## レンダリングパスの構成

```
Directional Light パス
  │
  ├─ VS: DeferredLightVertexShader（フルスクリーントライアングル）
  └─ PS: DeferredLightPixelShaders.usf（LIGHT_TYPE_DIRECTIONAL）
      1. GBuffer デコード
      2. シャドウマスク計算（PCF / PCSS / VSM）
      3. BRDF（GGX Specular + Lambert Diffuse）
      4. Sky Atmosphere 大気散乱の乗算（有効時）
      5. SceneColor にアディティブブレンド
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `GBufferA/B/C/D` | マテリアル属性（GBuffer デコード）|
| `ShadowDepthTexture` | シャドウデプスマップ（Cascaded Shadow Maps）|
| `LightAttenuation` | Shadow Mask テクスチャ |
| `SkyAtmosphere.TransmittanceLUT` | 大気透過率（Sky Atmosphere 有効時）|

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| `SceneColor` | `R11G11B10F` | ディレクショナルライトの Diffuse + Specular |

---

## シェーダーコアロジック

### GetDynamicLighting（DeferredLightingCommon.ush）

```hlsl
// LIGHT_TYPE_DIRECTIONAL 時のディレクショナルライト評価
FDeferredLightingSplit GetDynamicLighting(
    float3 TranslatedWorldPosition,
    float3 CameraVector,
    FGBufferData GBuffer,
    float AmbientOcclusion,
    uint ShadingModelID,
    FDeferredLightData LightData,
    float4 LightAttenuation,
    ...)
{
    // ライト方向（ディレクショナルは定数）
    float3 L = LightData.Direction;

    // 距離減衰なし（ディレクショナルライトは無限遠）
    float Attenuation = 1.0;

    // BRDF 評価
    FDirectLighting Lighting = EvaluateBRDF(GBuffer, CameraVector, L, Attenuation, LightData);

    // シャドウ × ライティング
    float Shadow = LightAttenuation.r;
    // [AtmosphereTransmittance]
    float3 AtmosphereTransmittance = GetAtmosphereTransmittance(WorldPosition, L);

    return Lighting * Shadow * AtmosphereTransmittance;
}
```

### PCF シャドウ（ShadowProjectionCommon.ush）

```hlsl
// Cascaded Shadow Maps の PCF サンプリング
float ManualPCF(float2 ShadowTexCoord, float ShadowDepth, float2 ShadowTexelSize)
{
    // 2〜16 サンプルの PCF（SHADOW_QUALITY に依存）
    float Shadow = 0;
    for (int i = 0; i < NUM_PCF_SAMPLES; i++)
    {
        float2 Offset = PCF_OFFSETS[i] * ShadowTexelSize;
        Shadow += ShadowDepthTexture.SampleCmpLevelZero(ShadowDepthSampler,
            ShadowTexCoord + Offset, ShadowDepth);
    }
    return Shadow / NUM_PCF_SAMPLES;
}
```

---

## CPU 呼び出しの流れ

```
FDeferredShadingSceneRenderer::RenderLight()     // LightRendering.cpp
  │
  ├─ ライトごとに 1 パスを生成
  ├─ [ディレクショナルライト]
  │   AddPass(RDG_EVENT_NAME("DirectionalLight"))
  │   VS: DeferredLightVertexShader
  │   PS: DeferredLightPixelShaders.usf（LIGHT_SOURCE_DIRECTIONAL）
  │   シャドウマスクは事前に RenderShadowProjections() で生成済み
  │
  └─ [Sky Atmosphere 統合]
      AtmosphereTransmittanceLUT を FFogUniformParameters 経由でバインド
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `DeferredLightPixelShaders.usf` | PS | ディレクショナル / ローカルライトの統一シェーダー |
| `DeferredLightingCommon.ush` | ヘッダ | `GetDynamicLighting()` / BRDF / シャドウ統合 |
| `ShadowProjectionCommon.ush` | ヘッダ | PCF / PCSS シャドウサンプリング |
| `SkyAtmosphereCommon.ush` | ヘッダ | 大気透過率の取得 |
