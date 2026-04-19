# GPU MegaLights ソースマップ

- 対象: MegaLights GPU シェーダー（タイル分類 → サンプリング → 可視性 → シェーディング → デノイズ → Volume）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_megalights_gpu_overview]]

確率的ライトサンプリング（BlueNoise + 重要度）→ SS/SW/HW RT または VSM で可視性 → BRDF 評価 → Temporal/Spatial デノイズ。
TILE_MODE 別に DownsampledTileAllocator / TileAllocator を分けて Indirect Dispatch。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Shaders/Private/MegaLights/` |
| シェーダー定数 | `Engine/Shared/MegaLightsDefinitions.h` |
| CPU | `Renderer/Private/MegaLights/MegaLights.cpp` 他 |

---

## ファイル → シェーダー対応

### タイル分類 / サンプリング

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `MegaLights.usf` | `MegaLightsTileClassificationBuildListsCS()` | `RenderMegaLightsViewContext()` | タイル分類 + Allocator 書き込み | [[detail_composite]] |
| `MegaLightsSampling.usf` | `GenerateLightSamplesCS()` | `FGenerateLightSamplesCS` | BlueNoise + 重要度サンプリング → LightSamples | [[detail_light_sampling]] |

### 可視性（シャドウ）

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `MegaLightsRayTracing.usf` | `ScreenSpaceRayTraceLightSamplesCS()` / `SoftwareRayTraceLightSamplesCS()` | SS/SW RT 経路 | 画面空間/ソフトウェア RT | [[detail_light_sampling]] |
| `MegaLightsHardwareRayTracing.usf` | HW RayGen | HW RT 経路（`MegaLights::RayTraceLightSamples()`） | DXR/Vulkan RT | 同 |

### シェーディング / デノイズ

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `MegaLightsShading.usf` | `ShadeLightSamplesCS()` | `FMegaLightsResolveCS` | BRDF 評価 → ResolvedDiffuse/Specular | [[detail_denoising]] |
| `MegaLightsDenoiserTemporal.usf` | `DenoiserTemporalCS()` | Temporal Filter | History ブレンド | 同 |
| `MegaLightsDenoiserSpatial.usf` | `DenoiserSpatialCS()` | Spatial Filter | 空間フィルタ → SceneColor 加算 | 同 |

### ボリューム（半透明用）

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `MegaLightsVolumeSampling.usf` | `VolumeGenerateLightSamplesCS()` | Volume 有効時 | ボリュームライトサンプリング | [[detail_composite]] |
| `MegaLightsVolumeRayTracing.usf` | `VolumeCompactLightSampleTracesCS()` / `VolumeSoftwareRayTraceLightSamplesCS()` | 同上 | ボリューム RT 可視性 | 同 |
| `MegaLightsVolumeShading.usf` | `VolumeShadeLightSamplesCS()` | 同上 | TranslucencyAmbient/Directional 書き込み | 同 |

---

## GPU データフロー

```
[A] TileClassification              MegaLights.usf:MegaLightsTileClassificationBuildListsCS
                                    → DownsampledTileAllocator / TileAllocator（TILE_MODE 別）

[B] LightSampling                   MegaLightsSampling.usf:GenerateLightSamplesCS
                                    → LightSamples / LightSampleRays (Texture2DArray)

[C] Visibility                      [EnabledRT ] HW RT: MegaLightsHardwareRayTracing.usf
                                    [EnabledRT ] SW/SS: MegaLightsRayTracing.usf:ScreenSpaceRayTraceLightSamplesCS / SoftwareRayTraceLightSamplesCS
                                    [EnabledVSM] MegaLights.cpp:MarkVSMPages

[D] Shading                         MegaLightsShading.usf:ShadeLightSamplesCS
                                    → ResolvedDiffuseLighting / ResolvedSpecularLighting

[E] Denoise                         MegaLightsDenoiserTemporal.usf:DenoiserTemporalCS
                                    MegaLightsDenoiserSpatial.usf:DenoiserSpatialCS
                                    → SceneColor 加算

[F] Volume (半透明用)               MegaLightsVolumeSampling.usf:VolumeGenerateLightSamplesCS
                                    MegaLightsVolumeRayTracing.usf:VolumeSoftwareRayTraceLightSamplesCS
                                    MegaLightsVolumeShading.usf:VolumeShadeLightSamplesCS
                                    → TranslucencyAmbient[TVC_MAX] / TranslucencyDirectional[TVC_MAX]
```

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `MegaLights.ush` | 共通型・定数（FMegaLightsData） |
| `MegaLightsMaterial.ush` | LoadMaterial, FMegaLightsMaterial |
| `MegaLightsSampling.ush` | FLightSampler 等 |
| `MegaLightsRayTracing.ush` | RT 共通 |
| `MegaLightsTileClassification.ush` | タイル分類定数 |
| `MegaLightsVolume.ush` | ボリューム座標変換 |
| `/Engine/Shared/MegaLightsDefinitions.h` | `TILE_MODE_*` 定数 |

---

## 主要テクスチャ

| リソース | 型 | 内容 |
|--------|-----|------|
| `LightSamples` | Texture2DArray | サンプル ID・重み・可視性 |
| `LightSampleRays` | Texture2DArray | シャドウレイ方向・距離 |
| `ResolvedDiffuseLighting` | Texture2D half3 | BRDF 評価済み拡散光 |
| `ResolvedSpecularLighting` | Texture2D half3 | BRDF 評価済み鏡面光 |
| `DiffuseLightingHistory` | Texture2D half3 | テンポラル蓄積済み拡散光 |
| `TranslucencyAmbient[TVC_MAX]` | Texture3D | 半透明 SH ボリューム |

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_light_sampling]] / [[detail_denoising]] / [[detail_composite]] |
| Reference | [[ref_light_sampling]] / [[ref_denoising]] / [[ref_composite]] |

---

## ue5-dive 起点

- 「タイル分類」 → `MegaLights.usf:MegaLightsTileClassificationBuildListsCS`
- 「ライトサンプリング」 → `MegaLightsSampling.usf:GenerateLightSamplesCS`
- 「HW RT 経路」 → `MegaLightsHardwareRayTracing.usf`
- 「BRDF 評価」 → `MegaLightsShading.usf:ShadeLightSamplesCS`
- 「デノイズ」 → `MegaLightsDenoiserTemporal/Spatial.usf`
- 「Volume ライティング」 → `MegaLightsVolumeShading.usf:VolumeShadeLightSamplesCS`
