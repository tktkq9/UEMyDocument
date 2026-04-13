# SceneRenderer 全体概要

- 取得日: 2026-04-13
- 対象: `Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.h/.cpp`
- 上位: [[01_rendering_overview]]

---

## SceneRenderer とは

**`FDeferredShadingSceneRenderer`** は UE5 のデフォルトレンダラー。  
毎フレーム `Render()` を呼ばれ、可視性カリングから Post-processing まで  
すべての描画パスを **RDG（Render Dependency Graph）** に登録して実行する。

```
GameThread               RenderThread               RHI Thread / GPU
─────────────────────    ─────────────────────────  ─────────────────
UWorld::Tick()           FDeferredShadingSceneRenderer::Render()
  └─ SendRenderTransform   │
     SendAllEndOfFrameUpdates
     └─ FSceneInterface::Render()
                          ├─ CommitFinalPipelineState()
                          ├─ BeginInitViews()
                          ├─ RenderPrePass / Nanite
                          ├─ RenderBasePass
                          ├─ Lighting / Lumen
                          ├─ PostProcessing
                          └─ FRDGBuilder::Execute() ──────────────▶ GPU 実行
```

---

## クラス階層

```
FSceneRendererBase
  └─ FSceneRenderer                    SceneRendering.h:2079
      │  Views[], Scene, ViewFamily
      │  InitViews() / RenderFinish()
      │
      └─ FDeferredShadingSceneRenderer  DeferredShadingRenderer.h:316
             Render() — 全パスのオーケストレーター
             CommitFinalPipelineState()
             FPerViewPipelineState / FFamilyPipelineState
```

---

## Render() 全体フロー

```
FDeferredShadingSceneRenderer::Render()             DeferredShadingRenderer.cpp:1736
  │
  ├─ OnRenderBegin()                                :1790  ビュー初期化タスク起動
  ├─ CommitFinalPipelineState()                     :1820  パイプライン状態確定（全ビュー）
  │
  ├─ [非同期タスク起動]
  │   ├─ VirtualShadowMapArray.Initialize()         :1869
  │   ├─ BeginUpdateLumenSceneTasks()               :1873
  │   └─ BeginGatherLumenLights()                   :1876
  │
  ├─ FSceneTextures::InitializeViewFamily()         :2046  シーンテクスチャ確保
  ├─ BeginInitViews()                               :2052  可視性計算（並列タスク開始）
  ├─ GPUScene.UploadDynamicPrimitiveShaderData()    :2172  GPU シーン更新
  ├─ EndInitViews()                                 :2316  DrawList 確定
  │
  ├─ RenderPrepassAndVelocity()                     :2594
  │   ├─ RenderPrePass()                            :2384  Z プリパス
  │   └─ RenderNanite()                             :2405  Nanite カリング＋ラスタライズ
  │
  ├─ RenderLumenSceneLighting()                     :2899  Lumen Surface Cache 更新
  ├─ RenderBasePass()                               :2905  GBuffer 書き込み（非 Nanite）
  ├─ RenderDiffuseIndirectAndAmbientOcclusion()     :3265  Lumen GI / AO
  ├─ RenderLights()                                 :3314  Direct Lighting
  ├─ RenderMegaLights()                             :3318  MegaLights（有効時）
  ├─ RenderDeferredReflectionsAndSkyLighting()      :3339  反射・スカイライト
  ├─ RenderTranslucency()（水中）                   :3520
  ├─ RenderTranslucency()（水上）                   :3654
  ├─ AddPostProcessingPasses()                      :3943  TAA/TSR・Bloom・Tonemap
  └─ OnRenderFinish()                               :4000  後処理・クリーンアップ
```

---

## 主要コンポーネントと役割

| コンポーネント | 役割 | 詳細ドキュメント |
|--------------|------|----------------|
| `FDeferredShadingSceneRenderer` | 全パスのオーケストレーター | [[a_deferred_renderer]] |
| `FScene` | レンダースレッド側シーンデータ（プリミティブ・ライト） | [[b_scene]] |
| `FViewInfo` / `FSceneView` | ビュー情報・DrawList・可視性マップ | [[c_view]] |
| BeginInitViews / EndInitViews | 視錐台カリング・HZB・ComputeRelevance | [[d_visibility]] |
| `USceneCaptureComponent2D` | SceneCapture → 独立ビュー投入 | [[e_scene_capture]] |
| `FPrimitiveSceneProxy` | GameThread ↔ RenderThread 橋渡し | [[f_scene_proxy]] |

---

## パイプライン状態（CommitFinalPipelineState で確定）

| 状態 | 型 | 説明 |
|------|----|------|
| `DiffuseIndirectMethod` | `EDiffuseIndirectMethod` | GI 方式（Lumen / SSGI / Disabled） |
| `AmbientOcclusionMethod` | `EAmbientOcclusionMethod` | AO 方式（SSAO / RTAO 等） |
| `ReflectionsMethod` | `EReflectionsMethod` | 反射方式（Lumen / SSR） |
| `bNanite` | `bool` | Nanite 有効フラグ |
| `bRayTracing` | `bool` | RT 全般有効フラグ |
| `bHZBOcclusion` | `bool` | HZB オクルージョン |

---

## 主要ソースファイル

| ファイル | 行数 | 内容 |
|---------|------|------|
| `DeferredShadingRenderer.h` | ~1180 | FDeferredShadingSceneRenderer クラス定義 |
| `DeferredShadingRenderer.cpp` | ~4200 | Render() メイン実装 |
| `SceneRendering.h` | ~3137 | FSceneRenderer / FViewInfo |
| `ScenePrivate.h` | ~2874 | FScene 本体 |
| `SceneVisibility.cpp` | — | BeginInitViews / CullPrimitives |
| `SceneCaptureRendering.cpp` | — | SceneCapture Component のレンダリング |
| `PrimitiveSceneProxy.h/.cpp` | — | FPrimitiveSceneProxy 基底クラス |

---

## 関連リファレンス

| リファレンス | 内容 |
|------------|------|
| [[ref_deferred_shading_renderer]] | FDeferredShadingSceneRenderer 全メソッド・状態構造体 |
| [[ref_scene_renderer]] | FSceneRenderer 基底クラス |
| [[ref_scene]] | FScene 主要メンバ |
| [[ref_view_info]] | FViewInfo・FSceneView |
