# GPU DeferredLighting ソースマップ

- 対象: Deferred Lighting GPU シェーダー（Directional + Clustered + LightFunction + Reflection）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_deferred_lighting_gpu_overview]]

Directional は全画面パス、ローカルライトは Clustered Deferred（LightGrid 参照）、
LightFunction は Atlas に事前レンダリング、Sky/Reflection は ReflectionEnvironmentPS で処理。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/DeferredLightPixelShaders.usf` |
| シェーダー | `Engine/Shaders/Private/ClusteredDeferredShadingPixelShader.usf` |
| シェーダー | `Engine/Shaders/Private/LightFunctionPixelShader.usf` |
| シェーダー | `Engine/Shaders/Private/LightFunctionAtlas/LightFunctionAtlasRender.usf` |
| シェーダー | `Engine/Shaders/Private/ReflectionEnvironmentPixelShader.usf` |
| CPU | `Renderer/Private/DeferredShadingRenderer.cpp` / `LightRendering.cpp` |

---

## ファイル → シェーダー対応

### ライティング

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `DeferredLightPixelShaders.usf` | `DeferredLightPixelShader()` | `FDeferredShadingSceneRenderer::RenderLight()` | 全画面ディレクショナル + Punctual BRDF | [[detail_directional]] |
| `ClusteredDeferredShadingPixelShader.usf` | `ClusteredShadingPixelShader()` | `RenderClusteredDeferredLighting()` | LightGrid 参照 → 全ローカルライト一括合算（Substrate Fast/Single/Complex 分岐対応） | [[detail_point_spot]] |

### Light Function

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `LightFunctionPixelShader.usf` | `Main()` | `RenderLightFunctionForAtlas()` | ライト関数マテリアル評価 | [[detail_light_functions]] |
| `LightFunctionAtlas/LightFunctionAtlasRender.usf` | 複数 | `RenderLightFunctionAtlas()` | Atlas への焼き込み | 同 |

### Sky / Reflection

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `ReflectionEnvironmentPixelShader.usf` | 複数 | `RenderReflections()` / `RenderDiffuseIndirectAndAmbientOcclusion()` | ReflectionCapture からの Specular + SkyLight Diffuse | [[detail_sky_light]] |

---

## GPU データフロー

```
[1] Directional Light                DeferredLightPixelShaders.usf:DeferredLightPixelShader
    → Punctual BRDF + PCF シャドウ → SceneColor Additive

[2] Clustered Deferred（Local Lights）ClusteredDeferredShadingPixelShader.usf:ClusteredShadingPixelShader
    → LightGrid 参照 → 全ローカルライト合算（Fast/Single/Complex パーミュテーション）

[3] Light Function Atlas             LightFunctionPixelShader.usf:Main
                                       LightFunctionAtlasRender.usf
    → LightFunction Texture Atlas 焼き込み → ライティング時に乗算

[4] Reflection Environment           ReflectionEnvironmentPixelShader.usf
    → Specular（ReflectionCapture）+ Diffuse（SkyLight）
```

---

## 主要パーミュテーション

| マクロ | 用途 |
|--------|------|
| `SUBSTRATE_ENABLED + SUBSTRATE_TILETYPE=0/1/2` | Fast/Single/Complex タイル |
| `LIGHT_FUNCTION_ATLAS_ENABLED` | Atlas 使用 |
| `USE_SHADOW_CACHE` | シャドウキャッシュ |
| `USE_CLOUD_TRANSMITTANCE` | 雲の影 |

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_directional]] / [[detail_point_spot]] / [[detail_light_functions]] / [[detail_sky_light]] |
| Reference | [[ref_directional]] / [[ref_point_spot]] / [[ref_light_functions]] / [[ref_sky_light]] |

---

## ue5-dive 起点

- 「ディレクショナルライト PS」 → `DeferredLightPixelShaders.usf:DeferredLightPixelShader`
- 「Clustered Deferred」 → `ClusteredDeferredShadingPixelShader.usf:ClusteredShadingPixelShader`
- 「Light Function Atlas」 → `LightFunctionAtlas/LightFunctionAtlasRender.usf`
- 「Substrate タイル分岐」 → `SUBSTRATE_TILETYPE` パーミュテーション
- 「Reflection + SkyLight 合成」 → `ReflectionEnvironmentPixelShader.usf`
