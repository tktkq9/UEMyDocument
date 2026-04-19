# SceneRenderer ソースマップ

- 対象: `FDeferredShadingSceneRenderer` を起点とするシーンレンダラー全体
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[23_scene_renderer_overview]]

`Render()` が全レンダリングパスをオーケストレートする。本マップは
ソースファイル → クラス/関数 → 解析ドキュメントの対応表。

---

## ソースパス

| 対象 | パス |
|------|------|
| レンダラー本体 | `Engine/Source/Runtime/Renderer/Private/` |
| 公開ヘッダ | `Engine/Source/Runtime/Renderer/Public/` |
| Scene 公開インタフェース | `Engine/Source/Runtime/Engine/Public/SceneInterface.h` |

---

## エントリファイル → クラス対応

### レンダラー本体

| ファイル | 行数目安 | 主要クラス / 関数 | 役割 | 参照 |
|---------|--------|----------------|------|------|
| `Private/DeferredShadingRenderer.h` | ~1180 | `FDeferredShadingSceneRenderer` | Deferred レンダラー定義。Per-view/Family Pipeline State | [[Reference/ref_deferred_shading_renderer]] |
| `Private/DeferredShadingRenderer.cpp` | ~4200 | `Render()`:1736, `CommitFinalPipelineState()`:1820 | 全パスのオーケストレーター実装 | [[Details/a_deferred_renderer]] |
| `Private/SceneRendering.h` | ~3137 | `FSceneRenderer`, `FViewInfo`, `FSceneRenderBuilder` | 基底レンダラー + レンダースレッド拡張ビュー | [[Reference/ref_scene_renderer]], [[Reference/ref_view_info]] |
| `Private/SceneRendering.cpp` | — | `FSceneRenderer::InitViews()`, `RenderFinish()` | ビュー初期化 / フレーム終了 | [[Details/c_view]] |
| `Private/SceneRenderBuilder.h/cpp` | — | `FSceneRenderBuilder` | レンダーコマンド積み上げ・実行 | — |

### シーンコア

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `Private/ScenePrivate.h` | `FScene` | レンダースレッド側シーン（~2874 行） | [[Reference/ref_scene]] |
| `Private/ScenePrivate.cpp` | `FScene::AddPrimitive/RemovePrimitive` | プリミティブ・ライト登録 | [[Details/b_scene]] |
| `Public/SceneInterface.h` | `FSceneInterface` | GameThread 公開 API | — |
| `Private/SceneCore.h/cpp` | `FStaticMeshBatch`, `FLightPrimitiveInteraction` | 静的メッシュバッチ・光源相互作用 | — |

### プリミティブ・ライト

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `Private/PrimitiveSceneInfo.h/cpp` | `FPrimitiveSceneInfo` | プリミティブのレンダースレッド表現 | [[Details/f_scene_proxy]] |
| `Public/PrimitiveSceneProxy.h` | `FPrimitiveSceneProxy` | プロキシ基底（GameThread ↔ RenderThread） | [[Details/f_scene_proxy]] |
| `Private/PrimitiveSceneProxy.cpp` | `FPrimitiveSceneProxy::*` | プロキシ実装 | — |
| `Private/LightSceneInfo.h/cpp` | `FLightSceneInfo`, `FLightSceneProxy` | ライトのシーン情報 | [[Details/b_scene]] |

### 可視性・カリング

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `Private/SceneVisibility.cpp` | `FSceneRenderer::BeginInitViews()`, `ComputeViewVisibility()`, `ComputeRelevance()` | 視錐台カリング・HZB・Relevance 計算 | [[Details/d_visibility]] |
| `Private/SceneOcclusion.cpp` | `OcclusionQuery` 系 | HW オクルージョンクエリ | — |

### SceneCapture

| ファイル | 主要クラス/関数 | 役割 | 参照 |
|---------|--------------|------|------|
| `Private/SceneCaptureRendering.cpp` | `USceneCaptureComponent2D::CaptureScene`, `FSceneCaptureCustomRenderPass` | SceneCapture レンダリング | [[Details/e_scene_capture]] |
| `Public/Components/SceneCaptureComponent.h` | `USceneCaptureComponent`, `USceneCaptureComponent2D`, `USceneCaptureComponentCube` | SceneCapture Component 定義 | [[Details/e_scene_capture]] |

---

## Render() フェーズ → ファイル対応

`FDeferredShadingSceneRenderer::Render()`（`DeferredShadingRenderer.cpp`）内の主要パスとそのソース位置。

| 行 | フェーズ | 実装ファイル | サブフォルダ |
|----|--------|------------|------------|
| :1790 | `OnRenderBegin()` | `DeferredShadingRenderer.cpp` | — |
| :1820 | `CommitFinalPipelineState()` | `DeferredShadingRenderer.cpp` | — |
| :1869 | `VirtualShadowMapArray.Initialize()` | `VirtualShadowMaps/VirtualShadowMapArray.cpp` | [[VirtualShadowMaps/]] |
| :1873 | `BeginUpdateLumenSceneTasks()` | `Lumen/LumenSceneData.cpp` | [[Lumen/]] |
| :2046 | `FSceneTextures::InitializeViewFamily()` | `SceneTextures.cpp` | — |
| :2052 | `BeginInitViews()` | `SceneVisibility.cpp` | — |
| :2172 | `GPUScene.UploadDynamicPrimitiveShaderData()` | `GPUScene.cpp` | [[GPUScene/]] |
| :2316 | `EndInitViews()` | `SceneVisibility.cpp` | — |
| :2384 | `RenderPrePass()` | `DepthRendering.cpp` | [[DepthPrepass/]] |
| :2405 | `RenderNanite()` | `Nanite/NaniteCullRaster.cpp` | [[Nanite/]] |
| :2899 | `RenderLumenSceneLighting()` | `Lumen/LumenSceneLighting.cpp` | [[Lumen/]] |
| :2905 | `RenderBasePass()` | `BasePassRendering.cpp` | [[BasePass/]] |
| :3265 | `RenderDiffuseIndirectAndAmbientOcclusion()` | `Lumen/LumenDiffuseIndirect.cpp` | [[Lumen/]], [[DistanceField/]] |
| :3314 | `RenderLights()` | `LightRendering.cpp` | [[DeferredLighting/]] |
| :3318 | `RenderMegaLights()` | `MegaLights/MegaLights.cpp` | [[MegaLights/]] |
| :3339 | `RenderDeferredReflectionsAndSkyLighting()` | `Lumen/LumenReflections.cpp`, `ReflectionEnvironment.cpp` | [[Lumen/]], [[SkyAtmosphere/]] |
| :3520 | `RenderTranslucency()`（水中） | `TranslucentRendering.cpp` | [[Translucency/]] |
| :3654 | `RenderTranslucency()`（水上） | `TranslucentRendering.cpp` | [[Translucency/]] |
| :3943 | `AddPostProcessingPasses()` | `PostProcess/PostProcessing.cpp` | [[PostProcess/]] |
| :4000 | `OnRenderFinish()` | `DeferredShadingRenderer.cpp` | — |

---

## Pipeline State（CommitFinalPipelineState 決定要素）

`FPerViewPipelineState` / `FFamilyPipelineState` に格納される方式決定フラグ。

| 状態 | 型 | 定義 | 参照 |
|------|----|-----|------|
| `DiffuseIndirectMethod` | `EDiffuseIndirectMethod` | `DeferredShadingRenderer.h` | GI 方式（Lumen / SSGI / Disabled） |
| `AmbientOcclusionMethod` | `EAmbientOcclusionMethod` | 同 | AO 方式（SSAO / RTAO） |
| `ReflectionsMethod` | `EReflectionsMethod` | 同 | 反射（Lumen / SSR） |
| `bNanite` / `bRayTracing` / `bHZBOcclusion` | `bool` | 同 | 機能有効フラグ |

---

## Reference/Details 一覧

| 種別 | ファイル | 内容 |
|------|---------|------|
| Reference | [[Reference/ref_deferred_shading_renderer]] | FDeferredShadingSceneRenderer 全メソッド・状態 |
| Reference | [[Reference/ref_scene_renderer]] | FSceneRenderer 基底 |
| Reference | [[Reference/ref_scene]] | FScene 主要メンバ |
| Reference | [[Reference/ref_view_info]] | FViewInfo / FSceneView |
| Details | [[Details/a_deferred_renderer]] | Render() フロー詳細 |
| Details | [[Details/b_scene]] | FScene プリミティブ・ライト管理 |
| Details | [[Details/c_view]] | FViewInfo / ViewFamily |
| Details | [[Details/d_visibility]] | BeginInitViews / Relevance |
| Details | [[Details/e_scene_capture]] | SceneCapture Component 処理 |
| Details | [[Details/f_scene_proxy]] | FPrimitiveSceneProxy 詳細 |

---

## ue5-dive 起点

- 「新しいパスを Render() に追加するには？」 → `DeferredShadingRenderer.cpp:Render()` 該当フェーズ付近 + RDG パス登録
- 「プリミティブの追加処理は？」 → `ScenePrivate.cpp:FScene::AddPrimitive` → `PrimitiveSceneInfo.cpp`
- 「可視性判定の中身」 → `SceneVisibility.cpp:ComputeViewVisibility/ComputeRelevance`
- 「ビュー情報の構造」 → `SceneRendering.h:FViewInfo` + [[Reference/ref_view_info]]
