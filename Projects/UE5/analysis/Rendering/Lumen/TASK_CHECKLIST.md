# Lumen リファレンスドキュメント作成 タスクチェックリスト

作成目標: `Reference/` フォルダ以下に各ソースファイルのクラス・関数リファレンスを作成し、  
対応する `Details/` 記事にリンクを追加する。

---

## フォルダ構成（完成形）

```
Rendering/Lumen/
├── 02_lumen_overview.md
├── Details/
│   ├── a_lumen_surface_cache.md    ← リンク追加済み？ [x]
│   ├── b_lumen_scene_lighting.md   ← リンク追加済み？ [x]
│   ├── c_lumen_tracing.md          ← リンク追加済み？ [x]
│   ├── d_lumen_radiance_cache.md   ← リンク追加済み？ [x]
│   ├── e_lumen_diffuse_gi.md       ← リンク追加済み？ [x]
│   └── f_lumen_reflections.md      ← リンク追加済み？ [x]
├── Reference/
│   ├── Common/
│   ├── a_SurfaceCache/
│   ├── b_SceneLighting/
│   ├── c_Tracing/
│   ├── d_RadianceCache/
│   ├── e_DiffuseGI/
│   └── f_Reflections/
└── TASK_CHECKLIST.md（このファイル）
```

---

## グループ共通（Common）— 3本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| C-1 | `Reference/Common/ref_lumen_core.md` | `Lumen.h` / `Lumen.cpp` | [x] 完了 |
| C-2 | `Reference/Common/ref_lumen_view_state.md` | `LumenViewState.h` | [x] 完了 |
| C-3 | `Reference/Common/ref_lumen_visualize.md` | `LumenVisualize.h/cpp` / `LumenVisualizeHardwareRayTracing.cpp` / `LumenVisualizeRadianceCache.cpp` | [x] 完了 |

---

## グループ a：Surface Cache — 7本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| A-1 | `Reference/a_SurfaceCache/ref_lumen_scene_data.md` | `LumenSceneData.h` | [x] 完了 |
| A-2 | `Reference/a_SurfaceCache/ref_lumen_mesh_cards.md` | `LumenMeshCards.h/cpp` | [x] 完了 |
| A-3 | `Reference/a_SurfaceCache/ref_lumen_surface_cache.md` | `LumenSurfaceCache.cpp` | [x] 完了 |
| A-4 | `Reference/a_SurfaceCache/ref_lumen_surface_cache_feedback.md` | `LumenSurfaceCacheFeedback.h/cpp` | [x] 完了 |
| A-5 | `Reference/a_SurfaceCache/ref_lumen_scene.md` | `LumenScene.cpp` | [x] 完了 |
| A-6 | `Reference/a_SurfaceCache/ref_lumen_scene_gpu_driven_update.md` | `LumenSceneGPUDrivenUpdate.h/cpp` | [x] 完了 |
| A-7 | `Reference/a_SurfaceCache/ref_lumen_utils.md` | `LumenSparseSpanArray.h` / `LumenUniqueList.h` | [x] 完了 |

---

## グループ b：Scene Lighting — 6本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| B-1 | `Reference/b_SceneLighting/ref_lumen_scene_lighting.md` | `LumenSceneLighting.h/cpp` | [x] 完了 |
| B-2 | `Reference/b_SceneLighting/ref_lumen_scene_card_capture.md` | `LumenSceneCardCapture.h/cpp` | [x] 完了 |
| B-3 | `Reference/b_SceneLighting/ref_lumen_scene_direct_lighting.md` | `LumenSceneDirectLighting.cpp` / `LumenSceneDirectLightingStochastic.inl` | [x] 完了 |
| B-4 | `Reference/b_SceneLighting/ref_lumen_scene_direct_lighting_hwrt.md` | `LumenSceneDirectLightingHardwareRayTracing.cpp` | [x] 完了 |
| B-5 | `Reference/b_SceneLighting/ref_lumen_radiosity.md` | `LumenRadiosity.h/cpp` | [x] 完了 |
| B-6 | `Reference/b_SceneLighting/ref_lumen_scene_rendering.md` | `LumenSceneRendering.cpp` | [x] 完了 |

---

## グループ c：Tracing — 6本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| C-1 | `Reference/c_Tracing/ref_lumen_tracing_utils.md` | `LumenTracingUtils.h/cpp` | [x] 完了 |
| C-2 | `Reference/c_Tracing/ref_lumen_mesh_sdf_culling.md` | `LumenMeshSDFCulling.cpp` | [x] 完了 |
| C-3 | `Reference/c_Tracing/ref_lumen_hwrt_common.md` | `LumenHardwareRayTracingCommon.h/cpp` | [x] 完了 |
| C-4 | `Reference/c_Tracing/ref_lumen_hwrt_materials.md` | `LumenHardwareRayTracingMaterials.cpp` | [x] 完了 |
| C-5 | `Reference/c_Tracing/ref_lumen_heightfields.md` | `LumenHeightfields.h/cpp` | [x] 完了 |
| C-6 | `Reference/c_Tracing/ref_lumen_irradiance_field.md` | `LumenIrradianceFieldGather.cpp` | [x] 完了 |

---

## グループ d：Radiance Cache — 5本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| D-1 | `Reference/d_RadianceCache/ref_lumen_radiance_cache.md` | `LumenRadianceCache.h/cpp` | [x] 完了 |
| D-2 | `Reference/d_RadianceCache/ref_lumen_radiance_cache_internal.md` | `LumenRadianceCacheInternal.h` / `LumenRadianceCacheInterpolation.h` | [x] 完了 |
| D-3 | `Reference/d_RadianceCache/ref_lumen_radiance_cache_hwrt.md` | `LumenRadianceCacheHardwareRayTracing.cpp` | [x] 完了 |
| D-4 | `Reference/d_RadianceCache/ref_lumen_translucency_radiance_cache.md` | `LumenTranslucencyRadianceCache.cpp` | [x] 完了 |
| D-5 | `Reference/d_RadianceCache/ref_lumen_translucency_volume.md` | `LumenTranslucencyVolumeLighting.h/cpp` / `LumenTranslucencyVolumeHardwareRayTracing.cpp` | [x] 完了 |

---

## グループ e：Diffuse GI — 7本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| E-1 | `Reference/e_DiffuseGI/ref_lumen_diffuse_indirect.md` | `LumenDiffuseIndirect.cpp` | [x] 完了 |
| E-2 | `Reference/e_DiffuseGI/ref_lumen_screen_probe_gather.md` | `LumenScreenProbeGather.h/cpp` | [x] 完了 |
| E-3 | `Reference/e_DiffuseGI/ref_lumen_screen_probe_tracing.md` | `LumenScreenProbeTracing.cpp` | [x] 完了 |
| E-4 | `Reference/e_DiffuseGI/ref_lumen_screen_probe_filtering.md` | `LumenScreenProbeFiltering.cpp` | [x] 完了 |
| E-5 | `Reference/e_DiffuseGI/ref_lumen_screen_probe_importance.md` | `LumenScreenProbeImportanceSampling.cpp` | [x] 完了 |
| E-6 | `Reference/e_DiffuseGI/ref_lumen_screen_probe_hwrt.md` | `LumenScreenProbeHardwareRayTracing.cpp` | [x] 完了 |
| E-7 | `Reference/e_DiffuseGI/ref_lumen_short_range_ao.md` | `LumenShortRangeAO.h` / `LumenShortRangeAOHardwareRayTracing.cpp` / `LumenScreenSpaceBentNormal.cpp` | [x] 完了 |

---

## グループ f：Reflections — 6本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| F-1 | `Reference/f_Reflections/ref_lumen_reflections.md` | `LumenReflections.h/cpp` | [x] 完了 |
| F-2 | `Reference/f_Reflections/ref_lumen_reflection_tracing.md` | `LumenReflectionTracing.cpp` | [x] 完了 |
| F-3 | `Reference/f_Reflections/ref_lumen_reflection_hwrt.md` | `LumenReflectionHardwareRayTracing.cpp` | [x] 完了 |
| F-4 | `Reference/f_Reflections/ref_lumen_restir.md` | `LumenReSTIRGather.h/cpp` | [x] 完了 |
| F-5 | `Reference/f_Reflections/ref_lumen_front_layer.md` | `LumenFrontLayerTranslucency.h/cpp` | [x] 完了 |
| F-6 | `Reference/f_Reflections/ref_ray_traced_translucency.md` | `RayTracedTranslucency.h` | [x] 完了 |

---

## 進捗サマリ

| グループ | 合計 | 完了 | 残り |
|---------|------|------|------|
| Common  | 3    | 3    | 0    |
| a: Surface Cache | 7 | 7 | 0 |
| b: Scene Lighting | 6 | 6 | 0 |
| c: Tracing | 6 | 6 | 0 |
| d: Radiance Cache | 5 | 5 | 0 |
| e: Diffuse GI | 7 | 7 | 0 |
| f: Reflections | 6 | 6 | 0 |
| **合計** | **40** | **40** | **0** |

---

## 作業ルール

1. 1セッションで **同グループをまとめて**依頼するのが効率的
2. リファレンス記事には以下を含める：
   - クラス一覧（継承・役割の一言説明）
   - 主要関数一覧（引数・戻り値・役割）
   - 主要マクロ（`SHADER_PARAMETER`系）
   - 関連ファイルリンク
3. Detail記事の末尾に `## 関連リファレンス` セクションを追加してリンクを張る
4. 完了したタスクは `[x]` に更新してコミット
