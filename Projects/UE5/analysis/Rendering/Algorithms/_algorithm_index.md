# Rendering アルゴリズム索引

UE5 Rendering で採用されているアルゴリズム / 数式 / PBR モデル / 近似手法と、その出典・実装を一覧化。

- 上位: [[../01_rendering_overview]]
- 関連: [[_source_index]] — 取得済み資料・未解決事項
- 手順書: [[../../HOWTO_create_algorithm_docs]]
- 更新日: 2026-04-26

---

## 1. BRDF / シェーディング

| 機能領域 | アルゴリズム | 出典 ID | 個別ドキュメント | UE 実装ファイル | ステータス |
|---------|-----------|--------|---------------|--------------|--------|
| NDF | GGX / Trowbridge-Reitz | S02 | [[brdf_ggx]] | `BRDF.ush:331` `D_GGX` | **完了 (2026-04-26)** |
| NDF | Anisotropic GGX | S03 | [[brdf_ggx_aniso]] | `BRDF.ush:339` `D_GGXaniso` | 未着手 |
| 幾何項 | Smith G2 (Joint Approx) | S04 | [[brdf_smith]] | `BRDF.ush:392` `Vis_SmithJointApprox` | **完了 (2026-04-26)** |
| 幾何項 | Schlick Visibility | S05 | [[brdf_schlick_vis]] | `BRDF.ush:374` `Vis_Schlick` | 未着手 |
| 幾何項 | Kelemen | S07 | [[brdf_kelemen]] | `BRDF.ush:366` `Vis_Kelemen` | 未着手 |
| Fresnel | Schlick Fresnel | S05 | [[brdf_fresnel_schlick]] | `BRDF.ush:423` `F_Schlick` | **完了 (2026-04-26)** |
| 拡散 | Disney Diffuse (Burley) | S03 | [[brdf_disney_diffuse]] | `BRDF.ush:183` `Diffuse_Burley` | **完了 (2026-04-26)** |
| 拡散 | Lambert | （古典） | [[brdf_lambert]] | `BRDF.ush:175` `Diffuse_Lambert` | 未着手 |
| IBL | Karis Split-Sum 近似 | S01 | [[brdf_split_sum]] | `BRDF.ush:611` `EnvBRDFApprox` | 未着手 |
| IBL | Lazarov Env BRDF Approx | S06 | [[brdf_lazarov_env]] | `BRDF.ush:630` `EnvBRDFApproxLazarov` | 未着手 |
| エリアライト | Linearly Transformed Cosines (LTC) | S08 | [[area_light_ltc]] | `RectLight.ush:334` | 未着手 |

## 2. グローバルイルミネーション

| 機能領域 | アルゴリズム | 出典 ID | 個別ドキュメント | UE 実装 | ステータス |
|---------|-----------|--------|---------------|--------|--------|
| Lumen | Surface Cache（カードベースの放射輝度キャッシュ） | S10 | [[lumen_surface_cache]] | `LumenSceneCardCapture.cpp` 等 | **完了 (2026-04-26)** |
| Lumen | Radiance Cache（球面ハーモニクス的な遠距離プローブ） | S10 | [[lumen_radiance_cache]] | `LumenRadianceCache.cpp` | **完了 (2026-04-26)** |
| Lumen | Software Ray Tracing（Mesh Distance Fields） | S10 | [[lumen_sw_rt]] | `LumenMeshSDFCulling.cpp` | **完了 (2026-04-26)** |
| Lumen | Hardware Ray Tracing | S10 | [[lumen_hw_rt]] | `LumenHardwareRayTracingCommon.cpp` | **完了 (2026-04-26)** |
| Lumen | Final Gather（ScreenProbeGather） | S10 | [[lumen_final_gather]] | `LumenScreenProbeGather.cpp` | **完了 (2026-04-26)** |
| Lumen | ReSTIR Gather | S11 | [[lumen_restir_gather]] | `LumenReSTIRGather.cpp` | **完了 (2026-04-26)** |

## 3. ジオメトリ

| 機能領域 | アルゴリズム | 出典 ID | 個別ドキュメント | UE 実装 | ステータス |
|---------|-----------|--------|---------------|--------|--------|
| Nanite | Hierarchical Cluster Culling（DAG） | S20 | [[nanite_cluster_culling]] | `NaniteCullRaster.cpp` | **完了 (2026-04-26)** |
| Nanite | Visibility Buffer Rendering | S20 | [[nanite_visibility_buffer]] | `NaniteMaterials.cpp` | **完了 (2026-04-26)** |
| Nanite | Software Rasterizer（Compute） | S20 | [[nanite_sw_raster]] | `NaniteRasterizer.usf` | **完了 (2026-04-26)** |
| Nanite | LOD 選択（screen-space error） | S20 | [[nanite_lod]] | `NaniteCullRaster.cpp` | **完了 (2026-04-26)** |

## 4. シャドウ

| 機能領域 | アルゴリズム | 出典 ID | 個別ドキュメント | UE 実装 | ステータス |
|---------|-----------|--------|---------------|--------|--------|
| VSM | Virtual Shadow Maps（Sparse Pages） | S21 | [[vsm_overview]] | `VirtualShadowMapArray.cpp` | **完了 (2026-04-26)** |
| VSM | Page Marking & Streaming | S21 | [[vsm_page_streaming]] | `VirtualShadowMapCacheManager.cpp` | **完了 (2026-04-26)** |
| シャドウ | PCF / PCSS フィルタ | S22 | [[shadow_pcf_pcss]] | `ShadowProjectionCommon.ush` | **完了 (2026-04-26)** |

## 5. アンチエイリアス / 解像度

| 機能領域 | アルゴリズム | 出典 ID | 個別ドキュメント | UE 実装 | ステータス |
|---------|-----------|--------|---------------|--------|--------|
| TAA | Karis TAA（Variance Clipping + YCoCg） | S30 | [[aa_taa]] | `TemporalAA.usf` | **完了 (2026-04-26)** |
| TSR | Temporal Super Resolution | S31 | [[aa_tsr]] | `TSR/` ディレクトリ | **完了 (2026-04-26)** |
| MSAA | Forward+ MSAA | （標準） | [[aa_msaa]] | （Forward Renderer） | 後回し |

## 6. スクリーンスペース

| 機能領域 | アルゴリズム | 出典 ID | 個別ドキュメント | UE 実装 | ステータス |
|---------|-----------|--------|---------------|--------|--------|
| SSAO | Mittring HBAO 系 | S40 | [[ss_ssao]] | `PostProcessAmbientOcclusion.cpp` | **完了 (2026-04-26)** |
| SSR | Stachowiak Screen Space Reflection | S41 | [[ss_ssr]] | `ScreenSpaceRayTracing.cpp` | **完了 (2026-04-26)** |
| SSGI | Screen Space Global Illumination | S42 | [[ss_ssgi]] | `ScreenSpaceRayTracing.cpp` | **完了 (2026-04-26)** |

## 7. ポストプロセス

| 機能領域 | アルゴリズム | 出典 ID | 個別ドキュメント | UE 実装 | ステータス |
|---------|-----------|--------|---------------|--------|--------|
| Tone Mapping | ACES（Academy Color Encoding System） | S50 | [[tone_aces]] | `ACES/ACESDisplayEncoding.ush` | **完了 (2026-04-26)** |
| Tone Mapping | Filmic（Hable / Reinhard） | S51 | [[tone_filmic]] | `TonemapCommon.ush` | **完了 (2026-04-26)** |
| Bloom | Mip-pyramid Convolution | S52 | [[post_bloom]] | `PostProcessBloomSetup.cpp` | **完了 (2026-04-26)** |
| Bloom | FFT Convolution（Lens Bloom） | S53 | [[post_bloom_fft]] | `PostProcessFFTBloom.cpp` | **完了 (2026-04-26)** |
| DOF | Bokeh Convolution | S54 | [[post_dof]] | `PostProcessDOF.cpp` | 未着手 |
| Motion Blur | Tile-based velocity scatter | S55 | [[post_motion_blur]] | `PostProcessMotionBlur.cpp` | 未着手 |

## 8. 大気・体積

| 機能領域 | アルゴリズム | 出典 ID | 個別ドキュメント | UE 実装 | ステータス |
|---------|-----------|--------|---------------|--------|--------|
| SkyAtmosphere | Hillaire 2020 物理大気 | S60 | [[atmos_hillaire]] | `SkyAtmosphereRendering.cpp` | **完了 (2026-04-26)** |
| Volumetric Cloud | Schneider Real-Time Clouds | S61 | [[atmos_clouds]] | `VolumetricCloudRendering.cpp` | **完了 (2026-04-26)** |
| Volumetric Fog | Frostbite Volumetric Fog | S62 | [[fog_volumetric]] | `VolumetricFog.cpp` | **完了 (2026-04-26)** |

## 9. SSS / 透過

| 機能領域 | アルゴリズム | 出典 ID | 個別ドキュメント | UE 実装 | ステータス |
|---------|-----------|--------|---------------|--------|--------|
| SSS | Burley Normalized SSS | S03 | [[sss_burley]] | `BurleyNormalizedSSSCommon.ush` | **完了 (2026-04-26)** |
| SSS | Separable SSS | S70 | [[sss_separable]] | `PostProcessSubsurface.cpp` | **完了 (2026-04-26)** |

---

## 凡例

- **出典 ID**: `_source_index.md` の S## 行に対応
- **ステータス**: `未着手` / `索引のみ` / `執筆中` / `完了`

---

## 関連ドキュメント

- [[../01_rendering_overview]] — Rendering 全体
- [[_source_index]] — 取得済み資料インデックス + 未解決事項
- [[../HOWTO_create_system_docs]] — システム文書手順
- [[../../HOWTO_create_algorithm_docs]] — Algorithms 文書手順
