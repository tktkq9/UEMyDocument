# RayTracing ソースマップ

- 対象: HW レイトレ基盤（TLAS/BLAS/SBT）+ RT Shadow/AO/SkyLight/Reflection/Translucency
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[07_raytracing_overview]]

`RHI_RAYTRACING` でガード。TLAS 構築（GatherInstances → Build）+ SBT（ヒットグループバインド）を毎フレーム行い、
Lumen HW RT / RT Shadow / AO / SkyLight / Reflection / Translucency の各パスで共用する。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/RayTracing/` |
| シェーダー | `Engine/Shaders/Private/RayTracing/*.usf` |
| 呼び出し元 | `DeferredShadingRenderer.cpp` |

---

## ファイル → クラス対応

### シーン構築（TLAS/BLAS）

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `RayTracing.h/.cpp` | `RayTracing::OnRenderBegin()`, `CreateGatherInstancesTaskData()`, `AddView()`, `BeginGatherInstances()`, `BeginGatherDynamicRayTracingInstances()`, `FinishGatherInstances()`, `FinishGatherVisibleShaderBindings()`, `FSceneOptions`, `FGatherInstancesTaskData` | 全体エントリ・インスタンス収集 | [[a_rt_scene]] |
| `RayTracingScene.h/.cpp` | `FRayTracingScene::Update()`, `FRayTracingScene::Build()` | TLAS RDG パス | 同 |
| `RayTracingInstanceCulling.h/.cpp` | `RayTracing::CullPrimitiveByFlags()`, `ShouldCullBounds()` | フラスタムカリング | 同 |
| `RayTracingInstanceMask.h/.cpp` | インスタンスマスク | ライティングチャンネル | 同 |

### SBT / マテリアル

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `RayTracingShaderBindingTable.h/.cpp` | `FRayTracingShaderBindingTable`, `FRayTracingShaderBindingData` | SBT 管理 | [[d_rt_materials_sbt]] |
| `RayTracingMaterialHitShaders.h/.cpp` | `SetRayTracingShaderBindings()`, `AddRayTracingLocalShaderBindingWriterTasks()` | マテリアルヒットシェーダー登録・RHI 反映 | 同 |
| `RayTracingLighting.h/.cpp` | RT ライティング共通 UB | — | — |

### 各 RT パス

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `RayTracingShadows.h/.cpp` | `RenderRayTracingShadows()` | RT シャドウ + デノイザー | [[b_rt_shadow_ao]] |
| `RayTracingAmbientOcclusion.cpp` | `RenderRayTracingAmbientOcclusion()` | RT AO | 同 |
| `RayTracingSkyLight.h/.cpp` | `RenderRayTracingSkyLight()` | RT スカイライト | 同 |
| `RayTracingPrimaryRays.cpp` | 1 次レイ（高品質反射等） | — | [[c_rt_reflection]] |
| `RayTracingTranslucency.cpp` | `RenderRayTracingTranslucency()` | 半透明 RT | 同 |
| `RayTracingDecals.h/.cpp` | デカールの RT 対応 | — | — |
| `RayTracingDynamicGeometryUpdateManager.cpp` | 動的ジオメトリ BLAS 更新 | スケルタル等 | [[a_rt_scene]] |
| `RaytracingOptions.h` | RT オプション構造体 | — | — |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─[1] RayTracing::OnRenderBegin                    RayTracing.cpp
  ├─[2] CreateGatherInstancesTaskData / AddView      ビューごとタスク初期化
  ├─[3] BeginGatherInstances                         並列: BLAS→TLAS 登録 + フラスタムカリング
  ├─[4] BeginGatherDynamicRayTracingInstances        動的 BLAS 更新（スケルタル等）
  ├─[5] FinishGatherInstances                        FRayTracingScene::Update → Build（TLAS RDG）
  ├─[6] FinishGatherVisibleShaderBindings            SBT 書き込み → SetRayTracingShaderBindings
  │
  ├─[7] RenderRayTracingShadows × 光源数             RayTracingShadows.cpp
  ├─[8] RenderRayTracingAmbientOcclusion             RayTracingAmbientOcclusion.cpp
  ├─[9] RenderRayTracingSkyLight                     RayTracingSkyLight.cpp
  ├─[10] RenderRayTracingReflections                 RayTracingReflections.cpp
  └─[11] RenderRayTracingTranslucency                RayTracingTranslucency.cpp

[Lumen HW RT] → 同じ TLAS を LumenHardwareRayTracing から参照
[MegaLights RT] → 同じ TLAS を MegaLightsRayTracing から参照
```

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.RayTracing` | （プロジェクト設定） |
| `r.RayTracing.Shadows` | `RayTracingShadows.cpp` |
| `r.RayTracing.AmbientOcclusion` | `RayTracingAmbientOcclusion.cpp` |
| `r.RayTracing.Reflections` | `RayTracingReflections.cpp` |
| `r.RayTracing.SkyLight` | `RayTracingSkyLight.cpp` |
| `r.RayTracing.ExcludeDecals` | `RayTracing.cpp` |
| `r.RayTracing.InstanceCulling` | `RayTracingInstanceCulling.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Details | [[a_rt_scene]] | TLAS/BLAS/GatherInstances/Culling |
| Details | [[b_rt_shadow_ao]] | RT Shadow / AO / SkyLight |
| Details | [[c_rt_reflection]] | RT Reflection / Translucency |
| Details | [[d_rt_materials_sbt]] | マテリアルヒットシェーダー / SBT |

---

## ue5-dive 起点

- 「RT のエントリ」 → `DeferredShadingRenderer.cpp` 内 `RayTracing::OnRenderBegin` 起点
- 「TLAS 構築」 → `RayTracingScene.cpp:FRayTracingScene::Update/Build`
- 「フラスタムカリング」 → `RayTracingInstanceCulling.cpp:CullPrimitiveByFlags/ShouldCullBounds`
- 「SBT（ヒットグループ）」 → `RayTracingShaderBindingTable.cpp` + `RayTracingMaterialHitShaders.cpp:SetRayTracingShaderBindings`
- 「Lumen HW RT との関係」 → Lumen は `LumenHardwareRayTracing.cpp` から同じ TLAS を参照
- 「Nanite の RT 対応」 → `Nanite/NaniteRayTracing.cpp:FRayTracingManager`（BLAS 構築キュー）
