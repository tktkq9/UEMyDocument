# GPU Lumen ソースマップ

- 対象: Lumen GPU シェーダー（Surface Cache + Screen Probe + Radiance Cache + Reflection + Composite）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_lumen_gpu_overview]]

Surface Cache（Card ベースオフスクリーンライティング）+ Screen Probe Gather（スクリーンスペース GI）の動的 GI。
Diffuse GI / Reflection は AsyncCompute で GBuffer と並列実行可能（`r.Lumen.AsyncCompute`）。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/Lumen/*.usf` |
| CPU | `Renderer/Private/Lumen/*.cpp` |

---

## ファイル → シェーダー対応

### Surface Cache（Card Capture）

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `LumenCardVertexShader.usf` / `LumenCardPixelShader.usf` | `Main()` | `LumenScene::RenderCardCaptureViews()` | [[a_SurfaceCache]] |
| `LumenCardComputeShader.usf` | `Main()` | 同上（Nanite 経路） | 同 |
| `SurfaceCache/LumenSurfaceCache.usf` | 複数 CS | `UpdateLumenScene()` | 同 |

### Scene Direct Lighting + Radiosity

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `LumenSceneDirectLightingCulling.usf` | `LumenSceneDirectLightingCullingCS()` | `RenderDirectLightingForLumenScene()` | [[b_SceneLighting]] |
| `LumenSceneDirectLightingShadowMask.usf` | `LumenSceneDirectLightingShadowMaskCS()` | 同上 | 同 |
| `LumenSceneDirectLighting.usf` | `LumenCardBatchDirectLightingCS()` | 同上 | 同 |
| `LumenSceneLighting.usf` | `CombineLumenSceneLightingCS()` | `RenderLumenSceneLighting()` | 同 |
| `Radiosity/LumenRadiosity.usf` | Mark/Trace/Filter/Integrate CS 群 | `RenderRadiosityForLumenScene()` | 同 |

### SDF トレーシング共有基盤

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `LumenMeshSDFCulling.usf` | `CullMeshSDFObjectsForViewCS()` / `CompactCulledObjectsCS()` | 各トレース関数 | [[c_Tracing]] |
| `LumenScene.usf` | `LumenCardUpdateCS()` 等 | `UpdateLumenScene()` | 同 |

### Radiance Cache

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `LumenRadianceCacheUpdate.usf` | `AllocateUsedProbesCS` / `AllocateProbeTracesCS` / `TraceFromProbesCS` / `FilterProbeRadianceWithGatherCS` / `CalculateProbeIrradianceCS` / `FixupBordersAndGenerateMipsCS` | `LumenRadianceCache::Update()` | [[d_RadianceCache]] |
| `LumenRadianceCache.usf` | `InterpolateRadianceCacheCS()` | `LumenRadianceCache::Render()` | 同 |

### Diffuse GI（Screen Probe Gather）

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `LumenScreenProbeTracing.usf` | `LumenScreenProbeTraceCS()` | `RenderLumenScreenProbeGather()` | [[e_DiffuseGI]] |
| `LumenScreenProbeGather.usf` | `LumenScreenProbeGatherCS()` | 同上 | 同 |
| `LumenScreenProbeGatherTemporal.usf` | `TemporallyAccumulateScreenProbesCS()` | 同上 | 同 |
| `LumenScreenProbeFiltering.usf` | `FilterScreenProbesCS()` | 同上 | 同 |
| `LumenReSTIRGather.usf` | `ReSTIRGatherCS` 他 | `RenderLumenReSTIRGather()` | 同 |
| `LumenScreenSpaceBentNormal.usf` | `LumenScreenSpaceBentNormalCS()` | `RenderLumenScreenSpaceBentNormal()` | 同 |

### Reflection

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `LumenReflections.usf` | `ReflectionTileClassificationBuildListsCS()` / `ReflectionGenerateRaysCS()` | `RenderLumenReflections()` | [[f_Reflections]] |
| `LumenReflectionTracing.usf` | `ReflectionTraceScreenTexturesCS()` / `ReflectionTraceMeshSDFsCS()` / `ReflectionTraceVoxelsCS()` | 同上 | 同 |
| `LumenReflectionResolve.usf` | `LumenReflectionResolveCS()` | 同上 | 同 |
| `LumenReflectionDenoiserTemporal.usf` | `LumenReflectionDenoiserTemporalCS()` | 同上 | 同 |
| `LumenReflectionDenoiserSpatial.usf` | `LumenReflectionDenoiserSpatialCS()` | 同上 | 同 |

### Final Composite

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `DiffuseIndirectComposite.usf` | `MainPS()` → `FDiffuseIndirectCompositePS` | `RenderDiffuseIndirectAndAmbientOcclusion()`（`IndirectLightRendering.cpp:977`） | [[g_FinalComposite]] |

---

## GPU データフロー

```
[Surface Cache 構築]
  [1] Card Capture                    LumenCardVertexShader / PixelShader / ComputeShader.usf
  [2] Direct Lighting                 LumenSceneDirectLightingCulling/ShadowMask/LumenSceneDirectLighting.usf
  [3] Radiosity                       Radiosity/LumenRadiosity.usf + LumenSceneLighting.usf:CombineLumenSceneLightingCS

[SDF トレース基盤]
  [4] CullMeshSDFObjectsForViewCS / CompactCulledObjectsCS（LumenMeshSDFCulling.usf）

[Radiance Cache]
  [5] LumenRadianceCacheUpdate.usf 群 → LumenRadianceCache.usf:InterpolateRadianceCacheCS

[Diffuse GI（AsyncCompute）]
  [6] LumenScreenProbeTracing → Gather → GatherTemporal → Filtering
  [7] (Option) LumenReSTIRGather / LumenScreenSpaceBentNormal

[Reflection（AsyncCompute）]
  [8] LumenReflections:ReflectionTileClassificationBuildListsCS → GenerateRaysCS
      → LumenReflectionTracing（Screen/MeshSDF/Voxel）
      → LumenReflectionResolve → DenoiserTemporal → DenoiserSpatial

[Final Composite]
  [9] DiffuseIndirectComposite.usf:MainPS → SceneColor 加算
```

---

## AsyncCompute

| キュー | パス |
|--------|------|
| Graphics | Card Capture, Surface Cache Lighting, Base Pass（GBuffer） |
| AsyncCompute | Screen Probe Gather, Reflections（`r.Lumen.AsyncCompute=1`）, Radiance Cache（`r.Lumen.RadianceCache.AsyncCompute=1`） |

---

## Details/Reference 一覧

| グループ | 詳細 | リファレンス |
|---------|------|-----------|
| a: SurfaceCache | [[detail_card_capture]] | [[ref_card_shaders]] |
| b: SceneLighting | [[detail_direct_lighting]] / [[detail_radiosity]] | [[ref_direct_lighting]] / [[ref_radiosity]] |
| c: Tracing | [[detail_sdf_tracing]] | [[ref_sdf_tracing]] |
| d: RadianceCache | [[detail_radiance_cache]] | [[ref_radiance_cache]] |
| e: DiffuseGI | [[detail_screen_probe_gather]] | [[ref_screen_probe_gather]] / [[ref_restir_bent_normal]] |
| f: Reflections | [[detail_reflections]] | [[ref_reflection_tracing]] / [[ref_reflection_denoiser]] |
| g: FinalComposite | [[detail_diffuse_indirect_composite]] | [[ref_diffuse_indirect_composite]] |

---

## ue5-dive 起点

- 「Lumen エントリ」 → `LumenVisualize.cpp` / `LumenSceneRendering.cpp`
- 「Card Capture」 → `LumenCardVertexShader.usf` + `LumenCardPixelShader.usf` / `LumenCardComputeShader.usf`
- 「Screen Probe Gather」 → `LumenScreenProbeTracing.usf` + `LumenScreenProbeGather.usf`
- 「Reflection」 → `LumenReflectionTracing.usf:ReflectionTraceMeshSDFsCS`
- 「最終合成」 → `DiffuseIndirectComposite.usf:MainPS`
- 「Async 経路」 → `r.Lumen.AsyncCompute` + `FD3D12Queue::WaitForOtherQueue`
