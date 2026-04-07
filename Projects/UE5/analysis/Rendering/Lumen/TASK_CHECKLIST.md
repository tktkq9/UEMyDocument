# Lumen リファレンスドキュメント作成 タスクチェックリスト

作成目標: `Reference/` フォルダ以下に各ソースファイルのクラス・関数リファレンスを作成し、  
対応する `Details/` 記事にリンクを追加する。

---

## フォルダ構成（完成形）

```
Rendering/Lumen/
├── 02_lumen_overview.md
├── Details/
│   ├── a_lumen_surface_cache.md    ← リンク追加済み？ [ ]
│   ├── b_lumen_scene_lighting.md   ← リンク追加済み？ [ ]
│   ├── c_lumen_tracing.md          ← リンク追加済み？ [ ]
│   ├── d_lumen_radiance_cache.md   ← リンク追加済み？ [ ]
│   ├── e_lumen_diffuse_gi.md       ← リンク追加済み？ [ ]
│   └── f_lumen_reflections.md      ← リンク追加済み？ [ ]
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
| A-1 | `Reference/a_SurfaceCache/ref_lumen_scene_data.md` | `LumenSceneData.h` | [ ] 未着手 |
| A-2 | `Reference/a_SurfaceCache/ref_lumen_mesh_cards.md` | `LumenMeshCards.h/cpp` | [ ] 未着手 |
| A-3 | `Reference/a_SurfaceCache/ref_lumen_surface_cache.md` | `LumenSurfaceCache.cpp` | [ ] 未着手 |
| A-4 | `Reference/a_SurfaceCache/ref_lumen_surface_cache_feedback.md` | `LumenSurfaceCacheFeedback.h/cpp` | [ ] 未着手 |
| A-5 | `Reference/a_SurfaceCache/ref_lumen_scene.md` | `LumenScene.cpp` | [ ] 未着手 |
| A-6 | `Reference/a_SurfaceCache/ref_lumen_scene_gpu_driven_update.md` | `LumenSceneGPUDrivenUpdate.h/cpp` | [ ] 未着手 |
| A-7 | `Reference/a_SurfaceCache/ref_lumen_utils.md` | `LumenSparseSpanArray.h` / `LumenUniqueList.h` | [ ] 未着手 |

---

## グループ b：Scene Lighting — 6本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| B-1 | `Reference/b_SceneLighting/ref_lumen_scene_lighting.md` | `LumenSceneLighting.h/cpp` | [ ] 未着手 |
| B-2 | `Reference/b_SceneLighting/ref_lumen_scene_card_capture.md` | `LumenSceneCardCapture.h/cpp` | [ ] 未着手 |
| B-3 | `Reference/b_SceneLighting/ref_lumen_scene_direct_lighting.md` | `LumenSceneDirectLighting.cpp` / `LumenSceneDirectLightingStochastic.inl` | [ ] 未着手 |
| B-4 | `Reference/b_SceneLighting/ref_lumen_scene_direct_lighting_hwrt.md` | `LumenSceneDirectLightingHardwareRayTracing.cpp` | [ ] 未着手 |
| B-5 | `Reference/b_SceneLighting/ref_lumen_radiosity.md` | `LumenRadiosity.h/cpp` | [ ] 未着手 |
| B-6 | `Reference/b_SceneLighting/ref_lumen_scene_rendering.md` | `LumenSceneRendering.cpp` | [ ] 未着手 |

---

## グループ c：Tracing — 6本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| C-1 | `Reference/c_Tracing/ref_lumen_tracing_utils.md` | `LumenTracingUtils.h/cpp` | [ ] 未着手 |
| C-2 | `Reference/c_Tracing/ref_lumen_mesh_sdf_culling.md` | `LumenMeshSDFCulling.cpp` | [ ] 未着手 |
| C-3 | `Reference/c_Tracing/ref_lumen_hwrt_common.md` | `LumenHardwareRayTracingCommon.h/cpp` | [ ] 未着手 |
| C-4 | `Reference/c_Tracing/ref_lumen_hwrt_materials.md` | `LumenHardwareRayTracingMaterials.cpp` | [ ] 未着手 |
| C-5 | `Reference/c_Tracing/ref_lumen_heightfields.md` | `LumenHeightfields.h/cpp` | [ ] 未着手 |
| C-6 | `Reference/c_Tracing/ref_lumen_irradiance_field.md` | `LumenIrradianceFieldGather.cpp` | [ ] 未着手 |

---

## グループ d：Radiance Cache — 6本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| D-1 | `Reference/d_RadianceCache/ref_lumen_radiance_cache.md` | `LumenRadianceCache.h/cpp` | [ ] 未着手 |
| D-2 | `Reference/d_RadianceCache/ref_lumen_radiance_cache_internal.md` | `LumenRadianceCacheInternal.h` / `LumenRadianceCacheInterpolation.h` | [ ] 未着手 |
| D-3 | `Reference/d_RadianceCache/ref_lumen_radiance_cache_hwrt.md` | `LumenRadianceCacheHardwareRayTracing.cpp` | [ ] 未着手 |
| D-4 | `Reference/d_RadianceCache/ref_lumen_translucency_radiance_cache.md` | `LumenTranslucencyRadianceCache.cpp` | [ ] 未着手 |
| D-5 | `Reference/d_RadianceCache/ref_lumen_translucency_volume.md` | `LumenTranslucencyVolumeLighting.h/cpp` / `LumenTranslucencyVolumeHardwareRayTracing.cpp` | [ ] 未着手 |

---

## グループ e：Diffuse GI — 7本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| E-1 | `Reference/e_DiffuseGI/ref_lumen_diffuse_indirect.md` | `LumenDiffuseIndirect.cpp` | [ ] 未着手 |
| E-2 | `Reference/e_DiffuseGI/ref_lumen_screen_probe_gather.md` | `LumenScreenProbeGather.h/cpp` | [ ] 未着手 |
| E-3 | `Reference/e_DiffuseGI/ref_lumen_screen_probe_tracing.md` | `LumenScreenProbeTracing.cpp` | [ ] 未着手 |
| E-4 | `Reference/e_DiffuseGI/ref_lumen_screen_probe_filtering.md` | `LumenScreenProbeFiltering.cpp` | [ ] 未着手 |
| E-5 | `Reference/e_DiffuseGI/ref_lumen_screen_probe_importance.md` | `LumenScreenProbeImportanceSampling.cpp` | [ ] 未着手 |
| E-6 | `Reference/e_DiffuseGI/ref_lumen_screen_probe_hwrt.md` | `LumenScreenProbeHardwareRayTracing.cpp` | [ ] 未着手 |
| E-7 | `Reference/e_DiffuseGI/ref_lumen_short_range_ao.md` | `LumenShortRangeAO.h` / `LumenShortRangeAOHardwareRayTracing.cpp` / `LumenScreenSpaceBentNormal.cpp` | [ ] 未着手 |

---

## グループ f：Reflections — 6本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| F-1 | `Reference/f_Reflections/ref_lumen_reflections.md` | `LumenReflections.h/cpp` | [ ] 未着手 |
| F-2 | `Reference/f_Reflections/ref_lumen_reflection_tracing.md` | `LumenReflectionTracing.cpp` | [ ] 未着手 |
| F-3 | `Reference/f_Reflections/ref_lumen_reflection_hwrt.md` | `LumenReflectionHardwareRayTracing.cpp` | [ ] 未着手 |
| F-4 | `Reference/f_Reflections/ref_lumen_restir.md` | `LumenReSTIRGather.h/cpp` | [ ] 未着手 |
| F-5 | `Reference/f_Reflections/ref_lumen_front_layer.md` | `LumenFrontLayerTranslucency.h/cpp` | [ ] 未着手 |
| F-6 | `Reference/f_Reflections/ref_ray_traced_translucency.md` | `RayTracedTranslucency.h` | [ ] 未着手 |

---

## 進捗サマリ

| グループ | 合計 | 完了 | 残り |
|---------|------|------|------|
| Common  | 3    | 3    | 0    |
| a: Surface Cache | 7 | 0 | 7 |
| b: Scene Lighting | 6 | 0 | 6 |
| c: Tracing | 6 | 0 | 6 |
| d: Radiance Cache | 5 | 0 | 5 |
| e: Diffuse GI | 7 | 0 | 7 |
| f: Reflections | 6 | 0 | 6 |
| **合計** | **40** | **0** | **40** |

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
