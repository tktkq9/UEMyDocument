# C: FSceneRenderer / FViewInfo — ビューとレンダラー基底

- 対象: `SceneRendering.h`
- 上位: [[01_rendering_overview]]
- Reference: [[ref_scene_renderer]] | [[ref_view_info]]

---

## クラス階層

```
FSceneRendererBase
  └─ FSceneRenderer                         SceneRendering.h:2079
      ├─ FDeferredShadingSceneRenderer       DeferredShadingRenderer.h:316
      └─ FMobileSceneRenderer               (モバイル用、別実装)

FSceneView                                  (SceneView.h — エンジン公開 API)
  └─ FViewInfo                              SceneRendering.h:1131
      └─ (レンダラー内部の拡張ビュー)
```

---

## FSceneRenderer — 主要メンバ変数

`SceneRendering.h:2079`

| 変数 | 型 | 説明 |
|-----|-----|------|
| `DepthPass` | `FDepthPassInfo` | Z プリパス設定（EarlyZPassMode 等） |
| `Allocator` | `FSceneRenderingBulkObjectAllocator` | レンダラーのライフタイムに紐付くアロケータ |
| `ViewFamily` | `FViewFamilyInfo` | ビューファミリー（SceneView.h の FSceneViewFamily 拡張） |
| `Views` | `TArray<FViewInfo>` | フレームで描画するビュー配列（通常 1、VR 時 2） |
| `AllViews` | `TArray<FViewInfo*>` | 通常ビュー + CustomRenderPass ビューすべて |
| `VisibleLightInfos` | `TArray<FVisibleLightInfo>` | 可視ライト情報（カリング後） |
| `VirtualShadowMapArray` | `FVirtualShadowMapArray` | 仮想シャドウマップ（[[04_vsm_overview]] 参照） |
| `LightFunctionAtlas` | `FLightFunctionAtlas` | ライト関数アトラス |
| `FeatureLevel` | `ERHIFeatureLevel::Type` | SM5 / SM6 等の RHI フィーチャーレベル |
| `ShaderPlatform` | `EShaderPlatform` | シェーダープラットフォーム |
| `bUsedPrecomputedVisibility` | `bool` | プリコンピュート可視性を使用したか |

### 主要仮想メソッド

```cpp
// 純粋仮想 — 派生クラスで実装される
virtual void Render(FRDGBuilder& GraphBuilder, const FSceneRenderUpdateInputs* SceneUpdateInputs) = 0;
virtual void RenderHitProxies(FRDGBuilder& GraphBuilder, const FSceneRenderUpdateInputs*) = 0;
virtual bool ShouldRenderVelocities() const = 0;
virtual bool ShouldRenderPrePass() const = 0;
virtual bool ShouldRenderNanite() const = 0;
```

### FSceneRenderer 生成フロー

```cpp
// SceneRenderBuilder.cpp 側で生成
FSceneRenderer* Renderer = FSceneRenderer::CreateSceneRenderer(ViewFamily, HitProxyConsumer);
// ↑ ViewFamily.Scene->GetShadingPath() に応じて
//   → FDeferredShadingSceneRenderer または FMobileSceneRenderer を生成
```

---

## FViewInfo — ビュー拡張クラス

`SceneRendering.h:1131`。`FSceneView`（エンジン公開 API）を継承し、  
レンダースレッド専用の可視性・DrawList 情報を追加している。

### 主要メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `ViewRect` | `FIntRect` | スクリーン上のビューピクセル矩形 |
| `ViewState` | `FSceneViewState*` | フレーム間で永続するビュー状態（TAA 用履歴テクスチャ等） |
| `PrimitiveVisibilityMap` | `FSceneBitArray` | 可視プリミティブのビットマップ |
| `PrimitiveViewRelevanceMap` | `TArray<FPrimitiveViewRelevance>` | プリミティブのパス参加フラグ |
| `DynamicMeshElements` | `TArray<FMeshBatchAndRelevance>` | 動的メッシュの DrawList |
| `VisibleLightInfos` | `TArray<FVisibleLightViewInfo>` | ビュー別の可視ライト情報 |
| `NumVisibleDynamicPrimitives` | `int32` | 可視動的プリミティブ数 |
| `TranslucentPrimCount` | `FTranslucenyPrimCount` | 透過プリミティブのカウンタ |
| `PrevViewInfo` | `FPreviousViewInfo` | 前フレームのビュー情報（TAA / モーションブラー用） |
| `HZBOcclusionTester` | `FHZBOcclusionTester` | HZB オクルージョンクエリ管理 |
| `GlobalDistanceFieldInfo` | `FGlobalDistanceFieldInfo` | Lumen / AO 用グローバル SDF |

### FPrimitiveViewRelevance — プリミティブのパス参加フラグ

```cpp
struct FPrimitiveViewRelevance
{
    uint8 bDrawRelevance      : 1;  // このビューで描画するか
    uint8 bStaticRelevance    : 1;  // スタティックメッシュパスを通るか
    uint8 bDynamicRelevance   : 1;  // ダイナミックメッシュパスを通るか
    uint8 bOpaque             : 1;  // 不透明か
    uint8 bTranslucentSurface : 1;  // 半透明か
    uint8 bShadowRelevance    : 1;  // シャドウパスを通るか
    uint8 bVelocityRelevance  : 1;  // ベロシティパスを通るか
    // ...
};
```

---

## FSceneViewState — フレーム間状態

`FViewInfo::ViewState` に保持される。TAA・SSR・Lumen 等がフレーム間データを共有するために使う。

```cpp
class FSceneViewState
{
    // TAA / TSR 用の履歴テクスチャ
    TRefCountPtr<IPooledRenderTarget> PrevSceneColor;
    TRefCountPtr<IPooledRenderTarget> TemporalAAHistoryRT;

    // Lumen 用のラジアンスキャッシュ
    FLumenViewState LumenViewState;

    // HZB オクルージョン
    TRefCountPtr<IPooledRenderTarget> HZB;
};
```

---

## コード実行フロー

### エントリポイント

```
FSceneRenderer::CreateSceneRenderer(ViewFamily, HitProxyConsumer)
  └─ new FDeferredShadingSceneRenderer(ViewFamily, HitProxyConsumer)
      ├─ FSceneRenderer コンストラクタ
      │   ├─ ViewFamily を FViewFamilyInfo でラップ
      │   ├─ 各 FSceneView → FViewInfo に変換して Views 配列に追加
      │   └─ AllViews を構築（Views + CustomRenderPassViews）
      └─ FDeferredShadingSceneRenderer コンストラクタ
          ├─ FPerViewPipelineState を Views.Num() 分確保
          └─ LumenCardRenderer を初期化

FDeferredShadingSceneRenderer::Render()
  └─ BeginInitViews()
      └─ 各 FViewInfo に対して:
          ├─ FSceneViewState から前フレーム情報をロード
          ├─ FrustumCull → PrimitiveVisibilityMap を構築
          └─ ComputeRelevance → PrimitiveViewRelevanceMap を構築
```

### フロー詳細

1. **FSceneView → FViewInfo の変換**
   ```cpp
   // FSceneRenderer コンストラクタ内
   for (int32 ViewIndex = 0; ViewIndex < ViewFamily->Views.Num(); ViewIndex++)
   {
       Views.Emplace(*ViewFamily->Views[ViewIndex]);
       // FSceneView の全メンバをコピーしつつ、
       // ViewRect / PrimitiveVisibilityMap 等の RenderThread 専用フィールドを初期化
   }
   ```

2. **ViewState とフレーム間データ**
   - `FViewInfo::ViewState` は `ULocalPlayer` が所有し、フレームをまたいで永続する
   - TAA 履歴テクスチャ・Lumen ラジアンスキャッシュ・HZB テクスチャを保持
   - `ViewState == nullptr` のビュー（SceneCapture 等）はフレーム間データを使わない

3. **AllViews の構築**
   ```cpp
   // 通常ビュー + CustomRenderPass ビューをフラット配列にまとめる
   for (FViewInfo& View : Views)          AllViews.Add(&View);
   for (auto& CRPI : CustomRenderPassInfos)
       for (FViewInfo& View : CRPI.Views) AllViews.Add(&View);
   ```

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|--------|------|
| `FSceneRenderer` | `SceneRendering.h:2079` | [[ref_scene_renderer]] レンダラー基底クラス |
| `FViewInfo` | `SceneRendering.h:1131` | [[ref_view_info]] レンダースレッド拡張ビュー |
| `FSceneView` | `SceneView.h` | エンジン公開 API のビューデータ |
| `FSceneViewState` | `SceneRendering.h` | フレーム間永続ビュー状態 |
| `FSceneRenderer::CreateSceneRenderer()` | `SceneRendering.cpp` | レンダラーファクトリ |
| `FPrimitiveViewRelevance` | `PrimitiveViewRelevance.h` | プリミティブのパス参加フラグ |
