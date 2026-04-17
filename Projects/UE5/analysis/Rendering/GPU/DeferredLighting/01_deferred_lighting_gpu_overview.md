# Deferred Lighting GPU 処理概要

- グループ: DeferredLighting GPU
- CPU 概要: [[11_deferred_lighting_overview]]
- CPU 詳細: [[a_directional_light]] | [[b_local_lights]] | [[c_light_functions]] | [[d_sky_light]]

---

## Deferred Lighting GPU パイプライン実行順

Deferred Lighting は GBuffer を読み取ってライティングを計算し SceneColor に加算する。  
ディレクショナルライトは全画面パス、ローカルライトは Clustered / Tiled 方式で処理する。

```
[ディレクショナルライト]
[1]  Directional Light（全画面パス）
     └─ DeferredLightPixelShader（DeferredLightPixelShaders.usf）
          GBuffer デコード → Punctual BRDF 計算 → SceneColor に加算

[ローカルライト（Clustered Deferred）]
[2]  Clustered Deferred Shading
     ├─ ClusteredShadingPixelShader（ClusteredDeferredShadingPixelShader.usf）
     │   LightGrid から適用ライトを取得 → GBuffer デコード → 全ローカルライトのライティング合算
     └─ [SubstrateTypeごと FastPath / SinglePath / ComplexPath に分岐]

[Light Functions]
[3]  Light Function Atlas / Per-Light Light Function
     ├─ LightFunctionPixelShader（LightFunctionPixelShader.usf）
     │   ライト関数マテリアルの評価 → Light Function Texture に焼き込み
     └─ LightFunctionAtlasRender（LightFunctionAtlas/LightFunctionAtlasRender.usf）

[Sky Light / Reflection]
[4]  Sky Light + Reflection Environment
     ├─ ReflectionEnvironmentPixelShader（ReflectionEnvironmentPixelShader.usf）
     │   ReflectionCapture キューブマップから Specular を評価
     └─ SkyLight DiffusePS（DiffuseIndirectComposite.usf 経由 / Lumen なし時）
```

---

## 各ステップの詳細

### [1] Directional Light

| 項目 | 内容 |
|-----|------|
| **概要** | 全画面フルスクリーンパスでディレクショナルライトを適用。PCF シャドウも統合 |
| **CPU 関数** | `FDeferredShadingSceneRenderer::RenderLight()` |
| **シェーダー** | PS: `DeferredLightPixelShaders.usf`（`DeferredLightPixelShader` 関数）|
| **出力** | `SceneColor`（アディティブブレンド）|
| **CPU 詳細** | [[a_directional_light]] |
| **GPU シェーダー詳細** | [[detail_directional]] / [[ref_directional]] |

---

### [2] Clustered Deferred（ローカルライト）

| 項目 | 内容 |
|-----|------|
| **概要** | Light Grid（クラスター割り当て済み）から各ピクセルに影響するローカルライトを一括評価 |
| **CPU 関数** | `RenderClusteredDeferredLighting()` |
| **シェーダー** | PS: `ClusteredDeferredShadingPixelShader.usf#ClusteredShadingPixelShader()` |
| **出力** | `SceneColor`（アディティブブレンド）|
| **CPU 詳細** | [[b_local_lights]] |
| **GPU シェーダー詳細** | [[detail_point_spot]] / [[ref_point_spot]] |

---

### [3] Light Functions

| 項目 | 内容 |
|-----|------|
| **概要** | ライト形状に投影するマテリアルテクスチャ。Atlas に事前レンダリングしてライティング時に乗算 |
| **CPU 関数** | `RenderLightFunctionForAtlas()` |
| **シェーダー** | PS: `LightFunctionPixelShader.usf` / `LightFunctionAtlasRender.usf` |
| **出力** | LightFunction Texture Atlas |
| **GPU シェーダー詳細** | [[detail_light_functions]] / [[ref_light_functions]] |

---

### [4] Sky Light / Reflection

| 項目 | 内容 |
|-----|------|
| **概要** | ReflectionCapture キューブマップと SkyLight から Specular / Diffuse を追加 |
| **CPU 関数** | `RenderDiffuseIndirectAndAmbientOcclusion()` / `RenderReflections()` |
| **シェーダー** | PS: `ReflectionEnvironmentPixelShader.usf` |
| **CPU 詳細** | [[d_sky_light]] |
| **GPU シェーダー詳細** | [[detail_sky_light]] / [[ref_sky_light]] |

---

## シェーダー別 CPU 対応一覧

| シェーダーファイル | 主要エントリポイント | CPU 関数 | グループ |
|----------------|----------------|---------|---------|
| `DeferredLightPixelShaders.usf` | `DeferredLightPixelShader()` | `RenderLight()` | [[a_PointSpotLight]] / [[b_DirectionalLight]] |
| `ClusteredDeferredShadingPixelShader.usf` | `ClusteredShadingPixelShader()` | `RenderClusteredDeferredLighting()` | [[a_PointSpotLight]] |
| `LightFunctionPixelShader.usf` | `Main()` | `RenderLightFunctionForAtlas()` | [[c_LightFunctions]] |
| `LightFunctionAtlas/LightFunctionAtlasRender.usf` | 複数 | `RenderLightFunctionAtlas()` | [[c_LightFunctions]] |
| `ReflectionEnvironmentPixelShader.usf` | 複数 | `RenderReflections()` | [[d_SkyLight]] |
