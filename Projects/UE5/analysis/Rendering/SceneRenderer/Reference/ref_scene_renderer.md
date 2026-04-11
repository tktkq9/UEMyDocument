# リファレンス：SceneRendering.h（FSceneRenderer）

- グループ: b - SceneRenderer
- 上位: [[01_rendering_overview]]
- 関連: [[ref_deferred_shading_renderer]] | [[ref_view_info]] | [[ref_scene]]
- ソース: `Engine/Source/Runtime/Renderer/Private/SceneRendering.h`
- ソース: `Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp`

## 概要

`FSceneRenderer` はすべてのシーンレンダラーの抽象基底クラス。  
`FDeferredShadingSceneRenderer`（PC / Console）と  
`FMobileSceneRenderer`（モバイル）が派生する。  
ビューファミリー・シーン・ライト・シャドウの共通データを保持する。

---

## FSceneRenderer

`SceneRendering.h:2079`

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `DepthPass` | `FDepthPassInfo` | Z プリパス設定（EarlyZPassMode・bDitheredLODTransitions 等） |
| `Allocator` | `FSceneRenderingBulkObjectAllocator` | レンダラーのライフタイムに紐付くバルクアロケータ |
| `ViewFamily` | `FViewFamilyInfo` | ビューファミリー情報（`FSceneViewFamily` の拡張） |
| `Views` | `TArray<FViewInfo>` | 描画するビュー配列（通常 1、VR 時 2） |
| `AllViews` | `TArray<FViewInfo*>` | Views + CustomRenderPass ビューのフラット配列 |
| `CustomRenderPassInfos` | `TArray<FCustomRenderPassInfo>` | シーンキャプチャ等のカスタムレンダーパス |
| `VisibleLightInfos` | `TArray<FVisibleLightInfo, SceneRenderingAllocator>` | 可視ライト情報（カリング後） |
| `DispatchedShadowDepthPasses` | `TArray<FParallelMeshDrawCommandPass*>` | 並列シャドウ深度パス |
| `SortedShadowsForShadowDepthPass` | `FSortedShadowMaps` | シャドウ深度パス用ソート済みシャドウマップ |
| `VirtualShadowMapArray` | `FVirtualShadowMapArray` | VSM 全体管理（[[04_vsm_overview]] 参照） |
| `LightFunctionAtlas` | `LightFunctionAtlas::FLightFunctionAtlas` | ライト関数アトラス |
| `DynamicResolutionFractions` | `DynamicRenderScaling::TMap<float>` | 動的解像度スケール（システム別） |
| `FeatureLevel` | `ERHIFeatureLevel::Type` | SM5 / SM6 等の RHI フィーチャーレベル |
| `ShaderPlatform` | `EShaderPlatform` | シェーダープラットフォーム |
| `bUsedPrecomputedVisibility` | `bool` | プリコンピュート可視性を使用したか |
| `bGPUMasksComputed` | `bool` | GPU マスクが計算済みか（MGPU） |

### 主要仮想メソッド

```cpp
// 純粋仮想 — 派生クラスで実装
virtual void Render(FRDGBuilder& GraphBuilder, const FSceneRenderUpdateInputs* SceneUpdateInputs) = 0;
virtual void RenderHitProxies(FRDGBuilder& GraphBuilder, const FSceneRenderUpdateInputs*) = 0;
virtual bool ShouldRenderVelocities() const = 0;
virtual bool ShouldRenderPrePass() const = 0;
virtual bool ShouldRenderNanite() const = 0;
```

### 静的ファクトリメソッド

```cpp
// ViewFamily の ShadingPath に応じて適切な派生クラスを生成する
static FSceneRenderer* CreateSceneRenderer(
    const FSceneViewFamily* InViewFamily,
    FHitProxyConsumer* HitProxyConsumer);
```

---

## FDepthPassInfo

`SceneRendering.h` 内で定義。Z プリパスの動作を制御する。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `EarlyZPassMode` | `EDepthDrawingMode` | DDM_None / DDM_NonMasked / DDM_AllOpaque / DDM_AllOpaqueNoVelocity 等 |
| `bEarlyZPassMovable` | `bool` | 移動可能オブジェクトを Z プリパスに含めるか |
| `bDitheredLODTransitionsSeamless` | `bool` | LOD トランジションをシームレスに行うか（Dither）|

### EDepthDrawingMode

```cpp
enum EDepthDrawingMode
{
    DDM_None                 = 0,  // Z プリパスなし
    DDM_NonMaskedOnly        = 1,  // Masked でないメッシュのみ
    DDM_AllOpaque            = 2,  // 全不透明メッシュ（Nanite 時に強制）
    DDM_AllOpaqueNoVelocity  = 3,  // 全不透明 + ベロシティは別パス
    DDM_AllOccluders         = 4,  // HZB オクルーダー付きの全不透明
};
```

---

## FCustomRenderPassInfo（内部構造体）

`SceneRendering.h:2092`。`SceneCapture` 等がメインレンダラーに付随するカスタムパスとして登録される。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `CustomRenderPass` | `FCustomRenderPassBase*` | カスタムパスの実装（SceneCapture 等） |
| `ViewFamily` | `FViewFamilyInfo` | カスタムパス用の独立したビューファミリー |
| `Views` | `TArray<FViewInfo>` | カスタムパス用ビュー |
| `NaniteBasePassShadingCommands` | `FNaniteShadingCommands` | Nanite シェーディングコマンド |

---

## FRendererViewDataManager

ビューデータ（シーンユニフォームバッファ等）を集中管理するオブジェクト。  
`GraphBuilder.AllocObject<FRendererViewDataManager>(...)` で生成され、  
`FInstanceCullingManager` と連携して GPU ドリブンカリングに使用される。

---

> [!note]- FSceneRenderingBulkObjectAllocator の用途
> `FSceneRenderer::Allocator` はレンダラーのライフタイムに紐付くスタックアロケータ。  
> `SceneRenderingAllocator` タグ付きのコンテナ（例: `TArray<T, SceneRenderingAllocator>`）は  
> このアロケータを使い、フレーム終了時に一括解放される。  
> 毎フレームの一時データに使うことでフレーム間のメモリ断片化を防ぐ。

> [!note]- IsHeadLink() — マルチビューレンダラーのリンクリスト
> ```cpp
> struct { FSceneRenderer* Head; FSceneRenderer* Next; } Link;
> bool IsHeadLink() const { return Link.Head == this; }
> ```
> 複数のレンダラーが 1 フレームで動作する場合（ Editor の複数ビューポート等）、  
> `Link.Head` がマスターレンダラーを指し、`Next` でチェーンされる。  
> `IsHeadLink()` が false のレンダラーは一部の更新処理をスキップする。
