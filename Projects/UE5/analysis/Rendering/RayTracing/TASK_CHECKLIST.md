# Ray Tracing ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\RayTracing\`  
**注意**: `RHI_RAYTRACING` マクロでガード。DXR / Vulkan RT 対応ハードウェアのみ有効。

---

## 概要ファイル

- [x] `07_raytracing_overview.md` — アーキテクチャ・TLAS/BLAS・フロー・主要クラス・CVar

---

## Phase 1: Details（4本）

- [x] `Details/a_rt_scene.md`            — TLAS / BLAS 構築・GatherInstances・フラスタムカリング
- [x] `Details/b_rt_shadow_ao.md`        — RT シャドウ・RT AO・RT スカイライト パス
- [x] `Details/c_rt_reflection.md`       — RT リフレクション（Primary Rays）・RT 半透明
- [x] `Details/d_rt_materials_sbt.md`    — マテリアルヒットシェーダー・SBT 管理・動的 BLAS 更新

---

## Phase 2: Reference（4本）

- [x] `Reference/ref_rt_scene.md`        — `FRayTracingScene` / `FSceneOptions` / `FGatherInstancesTaskData`
- [x] `Reference/ref_rt_sbt.md`          — `FRayTracingShaderBindingTable` / `FRayTracingShaderBindingData`
- [x] `Reference/ref_rt_shadows.md`      — `RayTracingShadows.h` / `RayTracingAmbientOcclusion.cpp` クラス群
- [x] `Reference/ref_rt_instances.md`    — `RayTracingInstanceCulling.h` / `RayTracingInstanceMask.h` / `RayTracingMaterialHitShaders.h`

---

## Phase 3: コード実行フロー追加

- [x] `07_raytracing_overview.md` — OnRenderBegin → GatherInstances → FinishGather → RT パス群 フロー追加
- [x] `Details/a_rt_scene.md`     — BeginGatherInstances → BeginGatherDynamic → FinishGather → TLAS 構築 フロー
- [x] `Details/b_rt_shadow_ao.md` — RayTracingShadows / RayTracingAmbientOcclusion 各パス内部フロー
- [x] `Details/c_rt_reflection.md`— RayTracingPrimaryRays / RayTracingTranslucency フロー
- [x] `Details/d_rt_materials_sbt.md` — FinishGatherVisibleShaderBindings → SBT 確定 フロー

---

## 進捗サマリ

| フェーズ | 合計 | 完了 | 残り |
|---------|------|------|------|
| 概要 | 1 | 1 | 0 |
| Phase 1: Details | 4 | 4 | 0 |
| Phase 2: Reference | 4 | 4 | 0 |
| Phase 3: Flow | 5 | 5 | 0 |
| **合計** | **14** | **14** | **0** |

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `RayTracing.h/.cpp` | インスタンス収集・TLAS 構築・全体エントリ |
| `RayTracingScene.h/.cpp` | FRayTracingScene（TLAS）管理 |
| `RayTracingShaderBindingTable.h/.cpp` | SBT（ヒットグループシェーダーバインド）管理 |
| `RayTracingInstanceCulling.h/.cpp` | フラスタムカリング・インスタンスフィルタリング |
| `RayTracingInstanceMask.h/.cpp` | インスタンスマスク（ライティングチャンネル）|
| `RayTracingMaterialHitShaders.h/.cpp` | マテリアルヒットシェーダーのコンパイル・登録 |
| `RayTracingShadows.h/.cpp` | RT シャドウパス |
| `RayTracingAmbientOcclusion.cpp` | RT AO パス |
| `RayTracingSkyLight.h/.cpp` | RT スカイライトパス |
| `RayTracingPrimaryRays.cpp` | 1次レイ（反射等の高品質モード）|
| `RayTracingTranslucency.cpp` | 半透明 RT |
| `RayTracingDynamicGeometryUpdateManager.cpp` | 動的ジオメトリ BLAS 更新管理 |
