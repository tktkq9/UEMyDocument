# リファレンス：DeferredShadingRenderer.h / DeferredShadingRenderer.cpp

- グループ: a - DeferredRenderer
- 上位: [[01_rendering_overview]]
- 関連: [[ref_scene_renderer]] | [[ref_scene]] | [[ref_view_info]]
- ソース: `Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.h`
- ソース: `Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp`

## 概要

UE5 のデフォルトレンダリングパイプラインを実装する具象レンダラー。  
`FSceneRenderer` を継承し、遅延シェーディングを基盤として  
Nanite / Lumen / VSM / PostProcess 等のすべてのサブシステムをオーケストレーションする。

---

## FDeferredShadingSceneRenderer

`DeferredShadingRenderer.h:316`

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `LumenCardRenderer` | `FLumenCardRenderer` | Lumen カードキャプチャ用メッシュコマンド管理 |
| `ViewPipelineStates` | `TArray<TPipelineState<FPerViewPipelineState>, TInlineAllocator<1>>` | ビュー別パイプライン状態 |
| `FamilyPipelineState` | `TPipelineState<FFamilyPipelineState>` | ファミリー単位のパイプライン状態 |
| `SeparateTranslucencyDimensions` | `FSeparateTranslucencyDimensions` | セパレート半透明の解像度 |
| `NaniteBasePassVisibility` | `FNaniteBasePassVisibility` | Nanite ベースパスの可視性クエリ結果 |

### 公開メソッド

```cpp
FDeferredShadingSceneRenderer(const FSceneViewFamily* InViewFamily, FHitProxyConsumer* HitProxyConsumer);

// ── パイプライン状態確定 ──
void CommitFinalPipelineState();    // FPerViewPipelineState / FFamilyPipelineState を確定
void CommitIndirectLightingState(); // 間接ライティング関連の状態を確定

// ── レンダリングエントリ ──
virtual void Render(FRDGBuilder& GraphBuilder, const FSceneRenderUpdateInputs* SceneUpdateInputs) override;
virtual void RenderHitProxies(FRDGBuilder& GraphBuilder, const FSceneRenderUpdateInputs* SceneUpdateInputs) override;

// ── 条件判定 ──
virtual bool ShouldRenderVelocities() const override;
virtual bool ShouldRenderPrePass() const override;
virtual bool ShouldRenderNanite() const override;

// ── パス実行（Render から呼ばれる主要なもの） ──
// 注: RenderBasePass / RenderTranslucency は static free function
void RenderPrePass(FRDGBuilder& GraphBuilder, TArrayView<FViewInfo> InViews, ...);
void RenderNanite(FRDGBuilder& GraphBuilder, TArrayView<FViewInfo> InViews, ...);
void RenderLights(FRDGBuilder& GraphBuilder, FSceneTextures& SceneTextures, ...);
void RenderDiffuseIndirectAndAmbientOcclusion(FRDGBuilder& GraphBuilder, ...);
void RenderDeferredReflectionsAndSkyLighting(FRDGBuilder& GraphBuilder, ...);
void RenderOcclusion(FRDGBuilder& GraphBuilder, ...);
bool RenderHzb(FRDGBuilder& GraphBuilder, ...);

// ── 初期化 ──
FInitViewTaskDatas OnRenderBegin(FRDGBuilder& GraphBuilder, const FSceneRenderUpdateInputs* SceneUpdateInputs);
void BeginInitViews(FRDGBuilder& GraphBuilder, ...);
void EndInitViews(FRDGBuilder& GraphBuilder, ...);
void BeginUpdateLumenSceneTasks(FRDGBuilder& GraphBuilder, FLumenSceneFrameTemporaries& Temporaries);
```

---

## FPerViewPipelineState（内部構造体）

`DeferredShadingRenderer.h:457`。`CommitFinalPipelineState()` で確定後は不変。

### メンバ変数

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

---

## FFamilyPipelineState（内部構造体）

`DeferredShadingRenderer.h:480`。ビューファミリー全体に適用されるフラグ。

### メンバ変数

| メンバ | 型 | 説明 |
|--------|-----|------|
| `bRayTracingShadows` | `bool` | レイトレーシングシャドウ有効か（`RHI_RAYTRACING` のみ存在） |
| `bRayTracing` | `bool` | レイトレーシング全般が有効か |
| `bNanite` | `bool` | Nanite が有効か |
| `bHZBOcclusion` | `bool` | HZB オクルージョンが有効か |

---

## FLumenCardRenderer

`DeferredShadingRenderer.h:87`。Lumen カードキャプチャ用メッシュコマンドを管理するヘルパークラス。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `CardPagesToRender` | `TArray<FCardPageRenderData, SceneRenderingAllocator>` | キャプチャが必要なカードページ |
| `NumCardTexelsToCapture` | `int32` | キャプチャ対象テクセル総数 |
| `MeshDrawCommands` | `FMeshCommandOneFrameArray` | 1 フレーム用メッシュドローコマンド |
| `MeshDrawPrimitiveIds` | `TArray<int32, SceneRenderingAllocator>` | ドローコマンドのプリミティブ ID |
| `ResampledCardCaptureAtlas` | `FResampledCardCaptureAtlas` | リサンプリングキャプチャアトラス |
| `bPropagateGlobalLightingChange` | `bool` | グローバルライティング変化を全カードに伝播 |
| `bHasAnyCardCopy` | `bool` | コピー（リサンプリング）カードが存在するか |

### メソッド

```cpp
void Reset();  // 毎フレーム冒頭でリセット
```

---

## 関連 enum

```cpp
// DeferredShadingRenderer.h で定義

enum class EDiffuseIndirectMethod
{
    Disabled,
    SSGI,   // Screen Space GI
    Lumen,  // Lumen GI
    Plugin, // サードパーティ GI プラグイン
};

enum class EAmbientOcclusionMethod
{
    Disabled,
    SSAO,   // Screen Space AO
    SSGI,   // SSGI が AO バッファも生成
    RTAO,   // Ray Traced AO
};

enum class EReflectionsMethod
{
    Disabled,
    SSR,    // Screen Space Reflections
    Lumen,  // Lumen Reflections
};
```

---

> [!note]- ELumenIndirectLightingSteps — Lumen GI の実行ステップ制御
> ```cpp
> enum class ELumenIndirectLightingSteps
> {
>     None            = 0,
>     ScreenProbeGather = 1u << 0,  // Screen Probe GI の収集
>     Reflections       = 1u << 1,  // Lumen 反射
>     Composite         = 1u << 3,  // 合成
>     All = ScreenProbeGather | Reflections | Composite
> };
> ```
> `FAsyncLumenIndirectLightingOutputs::StepsLeft` で進行状況を管理する。  
> ステップが完了するたびに対応フラグが落とされる。

> [!note]- CommitFinalPipelineState() の責務
> ```cpp
> // Render() の先頭近く（:1820）で全レンダラーに対して呼ばれる
> for (FSceneRenderer* Renderer : SceneUpdateInputs->Renderers)
> {
>     static_cast<FDeferredShadingSceneRenderer*>(Renderer)->CommitFinalPipelineState();
> }
> ```
> この関数が確定させる内容:
> - 各ビューの `DiffuseIndirectMethod` / `AmbientOcclusionMethod` / `ReflectionsMethod`
> - `FFamilyPipelineState::bNanite` / `bHZBOcclusion` / `bRayTracing`
> - `ShouldRenderNanite()` / `ShouldRenderPrePass()` の戻り値はこれ以降変わらない
