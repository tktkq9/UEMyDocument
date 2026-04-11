# A: FDeferredShadingSceneRenderer — 構造とパイプライン状態

- 対象: `DeferredShadingRenderer.h/.cpp`
- 上位: [[01_rendering_overview]]
- Reference: [[ref_deferred_shading_renderer]]

---

## クラス概要

```
FSceneRendererBase
  └─ FSceneRenderer                  (SceneRendering.h:2079)
      └─ FDeferredShadingSceneRenderer  (DeferredShadingRenderer.h:316)
```

`FDeferredShadingSceneRenderer` は UE5 のデフォルトレンダラー。  
コンストラクタでシーン・ビュー情報をセットアップし、`Render()` で全パスを実行する。

---

## パイプライン状態管理

### FPerViewPipelineState（ビュー別）

`DeferredShadingRenderer.h:457`。`CommitFinalPipelineState()` で確定後は不変。

| メンバ | 型 | 説明 |
|--------|-----|------|
| `DiffuseIndirectMethod` | `EDiffuseIndirectMethod` | GI 方式（Disabled / SSGI / Lumen / Plugin） |
| `DiffuseIndirectDenoiser` | `IScreenSpaceDenoiser::EMode` | GI デノイザー種別 |
| `AmbientOcclusionMethod` | `EAmbientOcclusionMethod` | AO 方式（Disabled / SSAO / SSGI / RTAO） |
| `ReflectionsMethod` | `EReflectionsMethod` | 反射方式（Disabled / SSR / Lumen） |
| `ReflectionsMethodWater` | `EReflectionsMethod` | 水面の反射方式 |
| `bComposePlanarReflections` | `bool` | 平面反射を合成するか |
| `bFurthestHZB` | `bool` | 最遠 HZB を生成するか |
| `bClosestHZB` | `bool` | 最近 HZB を生成するか |

### FFamilyPipelineState（ファミリー単位）

`DeferredShadingRenderer.h:480`。ビューファミリー全体に適用されるフラグ。

| メンバ | 型 | 説明 |
|--------|-----|------|
| `bRayTracingShadows` | `bool` | レイトレーシングシャドウ有効か（RHI_RAYTRACING のみ） |
| `bRayTracing` | `bool` | レイトレーシング全般が有効か |
| `bNanite` | `bool` | Nanite が有効か |
| `bHZBOcclusion` | `bool` | HZB オクルージョンが有効か |

### 関連 enum

```cpp
enum class EDiffuseIndirectMethod { Disabled, SSGI, Lumen, Plugin };
enum class EAmbientOcclusionMethod { Disabled, SSAO, SSGI, RTAO };
enum class EReflectionsMethod { Disabled, SSR, Lumen };
```

---

## パス別の有効条件

| パス | 有効条件（主要判定） |
|-----|------------------|
| PrePass | `ShouldRenderPrePass()` が true（Nanite は常に強制） |
| Nanite | `ShouldRenderNanite()` が true（bNanite フラグ） |
| Lumen GI | `DiffuseIndirectMethod == EDiffuseIndirectMethod::Lumen` |
| Lumen Reflections | `ReflectionsMethod == EReflectionsMethod::Lumen` |
| VSM | `UseVirtualShadowMaps()` && EngineShowFlags.DynamicShadows |
| Ray Tracing Shadows | `bRayTracingShadows` かつ `RHI_RAYTRACING` |
| HZB Occlusion | `bHZBOcclusion` |

---

## FLumenCardRenderer

Lumen カードキャプチャ用メッシュコマンドを保持するヘルパークラス。

| メンバ | 型 | 説明 |
|--------|-----|------|
| `CardPagesToRender` | `TArray<FCardPageRenderData>` | キャプチャが必要なカードページ |
| `NumCardTexelsToCapture` | `int32` | キャプチャ対象テクセル総数 |
| `MeshDrawCommands` | `FMeshCommandOneFrameArray` | 1 フレーム用メッシュドローコマンド |
| `bPropagateGlobalLightingChange` | `bool` | グローバルライティング変化を全カードに伝播 |
| `bHasAnyCardCopy` | `bool` | コピー（リサンプリング）カードが存在するか |

---

## レンダーターゲットの概念階層

```
FSceneTextures（SceneTextures.h）
  ├─ Depth          — シーン深度（PrePass で書き込み）
  ├─ GBufferA/B/C/D — マテリアル属性（BasePass で書き込み）
  ├─ SceneColor     — HDR 最終カラー
  └─ Velocity       — モーションベクター
```

---

## コード実行フロー

### エントリポイント

```
FDeferredShadingSceneRenderer::Render()                DeferredShadingRenderer.cpp:1736
  │
  ├─ OnRenderBegin()                                   :1790  ビュー初期化タスク起動
  ├─ CommitFinalPipelineState()                        :1820  パイプライン状態確定（全ビュー）
  │
  ├─ [前処理: 非同期タスク起動]
  │   ├─ VirtualShadowMapArray.Initialize()            :1869
  │   ├─ BeginUpdateLumenSceneTasks()                  :1873  Lumen 更新（非同期）
  │   └─ BeginGatherLumenLights()                      :1876  Lumen ライト収集（非同期）
  │
  ├─ FSceneTextures::InitializeViewFamily()            :2046  シーンテクスチャ確保
  ├─ BeginInitViews()                                  :2052  可視性計算開始
  ├─ GPUScene.UploadDynamicPrimitiveShaderData()       :2172  GPU シーン更新
  ├─ EndInitViews()                                    :2316  可視性計算完了
  │
  ├─ RenderPrepassAndVelocity()                        :2594
  │   ├─ RenderPrePass()                              :2384  Z プリパス（深度先行描画）
  │   └─ RenderNanite()                               :2405  Nanite カリング＋ラスタライズ
  │
  ├─ RenderLumenSceneLighting()                        :2899  Lumen シーンライティング
  ├─ RenderBasePass()                                  :2905  GBuffer 書き込み（非 Nanite メッシュ）
  ├─ RenderDiffuseIndirectAndAmbientOcclusion()        :3265  Lumen GI / SSGI / AO
  ├─ RenderLights()                                    :3314  Direct Lighting
  ├─ RenderDeferredReflectionsAndSkyLighting()         :3339  反射・スカイライト
  ├─ RenderTranslucency()（水中）                      :3520
  ├─ RenderTranslucency()（水上）                      :3654
  ├─ AddPostProcessingPasses()                         :3943  TAA/TSR / Bloom / Tonemapper
  └─ OnRenderFinish()                                  :4000  後処理・クリーンアップ
```

### フロー詳細

1. **OnRenderBegin() — `:1790`**
   - ビュー単位の CPU タスク（DrawList 構築・VisibilityTask）を起動
   - `FInitViewTaskDatas` を返す（その後のパスに渡される）

2. **CommitFinalPipelineState() — `:1820`**
   ```cpp
   for (FSceneRenderer* Renderer : SceneUpdateInputs->Renderers)
   {
       static_cast<FDeferredShadingSceneRenderer*>(Renderer)->CommitFinalPipelineState();
   }
   // ← これ以降 FPerViewPipelineState / FFamilyPipelineState は不変
   ```
   - 各ビューの `EDiffuseIndirectMethod` / `EReflectionsMethod` 等を決定する
   - `ShouldRenderNanite()` / `ShouldRenderPrePass()` はここで確定したフラグを参照する

3. **BeginInitViews / EndInitViews — `:2052` / `:2316`**
   - 可視性カリング（視錐台 / オクルージョン / HZB）をタスクグラフで並列実行
   - `EndInitViews()` でメッシュ DrawList が確定する

4. **RenderBasePass() — `:2905`**
   ```cpp
   RenderBasePass(*this, GraphBuilder, Views, SceneTextures, DBufferTextures,
       BasePassDepthStencilAccess, ForwardScreenSpaceShadowMaskTexture,
       InstanceCullingManager, bNaniteEnabled, NaniteShadingCommands, NaniteRasterResults);
   ```
   - 非 Nanite メッシュが GBuffer（A/B/C/D）に書き込む
   - Nanite の GBuffer 書き込みはこの前に `RenderNanite()` で完了している

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|--------|------|
| `FDeferredShadingSceneRenderer::Render()` | `DeferredShadingRenderer.cpp:1736` | メインエントリ |
| `FDeferredShadingSceneRenderer::CommitFinalPipelineState()` | `DeferredShadingRenderer.h:325` | パイプライン状態確定 |
| `FSceneTextures::InitializeViewFamily()` | `SceneTextures.cpp` | シーンテクスチャ確保 |
| `FDeferredShadingSceneRenderer::BeginInitViews()` | `SceneVisibility.cpp` | 可視性計算開始 |
| `FDeferredShadingSceneRenderer::RenderBasePass()` | `BasePassRendering.cpp` | GBuffer 書き込み |
| `FDeferredShadingSceneRenderer::RenderLights()` | `LightRendering.cpp` | Direct Lighting |
| `AddPostProcessingPasses()` | `PostProcessing.cpp` | TAA/TSR / ポストプロセス |
| `FLumenCardRenderer` | `DeferredShadingRenderer.h:87` | [[ref_deferred_shading_renderer]] Lumen カード管理 |
| `FPerViewPipelineState` | `DeferredShadingRenderer.h:457` | [[ref_deferred_shading_renderer]] ビュー別パイプライン状態 |
