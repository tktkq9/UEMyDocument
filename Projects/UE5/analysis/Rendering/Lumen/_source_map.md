# Lumen ソースマップ

- 対象: 動的 GI + 反射（Surface Cache + Screen Probe + Reflections）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[02_lumen_overview]]

Lumen は (1) Lumen Scene（Card / Surface Cache）、(2) Surface Cache Lighting、(3) Tracing、
(4) Screen Probe Gather、(5) Reflections、(6) 最終合成の 6 層構造。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/Lumen/` |
| シェーダー | `Engine/Shaders/Private/Lumen/*.usf` |
| 呼び出し元 | `Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp`, `IndirectLightRendering.cpp` |

---

## ファイル → クラス対応

### ① Lumen Scene / Surface Cache

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `Lumen.h/.cpp` | `ShouldRenderLumenDiffuseGI()`, `ShouldRenderLumenReflections()` | 有効化判定 | — |
| `LumenSceneData.h/.cpp` | `FLumenSceneData`, `FLumenCardScene`, `FLumenCard`, `FLumenPrimitiveGroup`, `FLumenMeshCards` | シーン表現（Card / MeshCards / Primitive Group） | [[Reference/ref_lumen_scene]] |
| `LumenSceneRendering.h/.cpp` | `BeginUpdateLumenSceneTasks()`:1969, `UpdateLumenScene()`:2490, `FLumenSceneFrameTemporaries` | フレーム初期化・GPU Scene 更新 | [[Reference/ref_lumen_scene_card_capture]], [[Reference/ref_lumen_scene_gpu_driven_update]], [[Details/a_lumen_surface_cache]] |

### ② Surface Cache Lighting

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `LumenSceneLighting.cpp` | `RenderLumenSceneLighting()`:217 | Direct + Radiosity → Atlas | [[Reference/ref_lumen_scene_lighting]], [[Details/b_lumen_scene_lighting]] |
| `LumenDirectLighting.cpp` | `RenderDirectLightingForLumenScene()` | DirectLightingAtlas 書き込み | 同 |
| `LumenRadiosity.cpp` | `RenderRadiosityForLumenScene()` | IndirectLightingAtlas 書き込み | [[Reference/ref_lumen_radiosity]] |
| `LumenSurfaceCacheFeedback.cpp` | `FLumenCardMeshProcessor` | Card キャプチャ MeshPass | — |

### ③ Tracing

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `LumenMeshSDF.cpp` | Mesh SDF トレース | Mesh 単位 SDF Intersect | [[Details/c_lumen_tracing]] |
| `LumenGlobalDistanceField.cpp` | Global SDF / Voxel トレース | 広域 SDF | 同 |
| `LumenHardwareRayTracing.cpp` | HW RT 経由のトレース | DXR / Inline | 同 |

### ④ Radiance Cache

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `LumenRadianceCache.h/.cpp` | `FRadianceCacheState`, `FRadianceCacheParameters` | 遠距離プローブキャッシュ | [[Details/d_lumen_radiance_cache]] |
| `LumenWorldSpaceProbes.cpp` | World Space Probe 配置 | プローブ階層 | 同 |

### ⑤ Diffuse GI（Screen Probe）

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `LumenScreenProbeGather.cpp` | `RenderLumenFinalGather()`:2094, `RenderLumenScreenProbeGather()`:2156 | Screen Probe 配置・トレース・フィルタ・Gather | [[Reference/ref_lumen_diffuse_indirect]], [[Reference/ref_lumen_screen_probe_gather]], [[Details/e_lumen_diffuse_gi]] |
| `LumenReSTIRGather.cpp` | `RenderLumenReSTIRGather()` | ReSTIR ベースの Gather | 同 |

### ⑥ Reflections

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `LumenReflections.cpp` | `RenderLumenReflections()`, `TraceReflections()` | Roughness 別トレース + ReSTIR | [[Reference/ref_lumen_reflections]], [[Reference/ref_lumen_reflection_tracing]], [[Details/f_lumen_reflections]] |
| `LumenHardwareRayTracedReflections.cpp` | `RenderLumenHardwareRayTracingReflections()` | HW RT 反射経路 | [[Reference/ref_lumen_reflection_hwrt]] |
| `LumenFrontLayerTranslucency.cpp` | `RenderLumenFrontLayerTranslucencyReflections()` | 透明前面反射 | — |

### ⑦ 合成

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `IndirectLightRendering.cpp` | `RenderDiffuseIndirectAndAmbientOcclusion()`:977 | Lumen GI + AO 合成 → SceneColor | [[Details/g_lumen_final_composite]] |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()                    DeferredShadingRenderer.cpp
  │
  ├─[InitViews]
  │   BeginUpdateLumenSceneTasks                             LumenSceneRendering.cpp:1969
  │     └─ 非同期 CPU タスク: UpdateLumenScenePrimitives →
  │        UpdateSurfaceCacheMeshCards → ProcessLumenSurfaceCacheRequests
  │
  ├─[GBuffer 描画前]
  │   UpdateLumenScene                                       LumenSceneRendering.cpp:2490
  │     └─ Card Atlas 再確保・ページテーブル Upload・GPU Driven Update
  │   RenderLumenSceneLighting                               LumenSceneLighting.cpp:217
  │     ├─ RenderDirectLightingForLumenScene → DirectLightingAtlas
  │     ├─ RenderRadiosityForLumenScene       → IndirectLightingAtlas
  │     └─ FinalLightingAtlas = Direct + Indirect
  │
  ├─[Lighting フェーズ]
  │   RenderDiffuseIndirectAndAmbientOcclusion               IndirectLightRendering.cpp:977
  │     │
  │     ├─ [DiffuseIndirectMethod == Lumen]
  │     │   RenderLumenFinalGather                           LumenScreenProbeGather.cpp:2094
  │     │     ├─ UseReSTIRGather → RenderLumenReSTIRGather
  │     │     └─ else           → RenderLumenScreenProbeGather:2156
  │     │
  │     └─ [ReflectionsMethod == Lumen]
  │         RenderLumenReflections                           LumenReflections.cpp
  │           ├─ UseHardwareRayTracedReflections → HW RT
  │           └─ else → TraceReflections (Mesh SDF → Global SDF → Resolve)
  │
  └─[透明]
      RenderLumenFrontLayerTranslucencyReflections           LumenFrontLayerTranslucency.cpp
```

---

## 主要 CVar（抜粋）

| CVar | 対応ファイル |
|------|------------|
| `r.Lumen.DiffuseIndirect.Allow` | `Lumen.cpp` |
| `r.Lumen.Reflections.Allow` | `LumenReflections.cpp` |
| `r.Lumen.HardwareRayTracing` | `LumenHardwareRayTracing.cpp` |
| `r.Lumen.ScreenProbeGather.*` | `LumenScreenProbeGather.cpp` |
| `r.Lumen.ReSTIRGather.Enable` | `LumenReSTIRGather.cpp` |
| `r.LumenScene.SurfaceCache.*` | `LumenSceneRendering.cpp` |
| `r.Lumen.Radiosity.*` | `LumenRadiosity.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_lumen_scene]] | FLumenSceneData |
| Reference | [[Reference/ref_lumen_scene_card_capture]] | Card キャプチャ |
| Reference | [[Reference/ref_lumen_scene_gpu_driven_update]] | GPU Driven Update |
| Reference | [[Reference/ref_lumen_surface_cache]] | Surface Cache Atlas |
| Reference | [[Reference/ref_lumen_scene_lighting]] | Surface Cache Lighting |
| Reference | [[Reference/ref_lumen_radiosity]] | Radiosity |
| Reference | [[Reference/ref_lumen_diffuse_indirect]] | RenderLumenFinalGather |
| Reference | [[Reference/ref_lumen_screen_probe_gather]] | Screen Probe Gather |
| Reference | [[Reference/ref_lumen_reflections]] | RenderLumenReflections |
| Reference | [[Reference/ref_lumen_reflection_tracing]] | TraceReflections |
| Reference | [[Reference/ref_lumen_reflection_hwrt]] | HW RT 反射 |
| Details | [[Details/a_lumen_surface_cache]] | Surface Cache 詳細 |
| Details | [[Details/b_lumen_scene_lighting]] | Surface Cache Lighting 詳細 |
| Details | [[Details/c_lumen_tracing]] | Mesh SDF / Global SDF / HW RT |
| Details | [[Details/d_lumen_radiance_cache]] | Radiance Cache |
| Details | [[Details/e_lumen_diffuse_gi]] | Screen Probe Gather 詳細 |
| Details | [[Details/f_lumen_reflections]] | Reflections 詳細 |
| Details | [[Details/g_lumen_final_composite]] | 最終合成 |

---

## ue5-dive 起点

- 「Lumen のエントリ」 → `DeferredShadingRenderer.cpp` + `BeginUpdateLumenSceneTasks` / `UpdateLumenScene` / `RenderDiffuseIndirectAndAmbientOcclusion`
- 「Card がどう作られるか」 → `LumenSceneRendering.cpp:UpdateLumenScene` + `FLumenMeshCards`
- 「Surface Cache のライティング」 → `LumenSceneLighting.cpp:RenderLumenSceneLighting`
- 「Screen Probe の配置」 → `LumenScreenProbeGather.cpp:RenderLumenScreenProbeGather`
- 「Reflections の経路分岐」 → `LumenReflections.cpp:RenderLumenReflections` + `UseHardwareRayTracedReflections`
- 「Lumen と非 Lumen の切替」 → `Lumen.h:ShouldRenderLumenDiffuseGI/Reflections`
