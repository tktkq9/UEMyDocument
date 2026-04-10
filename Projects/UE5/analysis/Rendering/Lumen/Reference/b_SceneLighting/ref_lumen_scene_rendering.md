# リファレンス：LumenSceneRendering.cpp

- グループ: b - Scene Lighting
- 上位: [[b_lumen_scene_lighting]]
- 関連: [[ref_lumen_scene_lighting]] | [[ref_lumen_scene_card_capture]] | [[ref_lumen_scene_data]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneRendering.cpp`

---

## 概要

Lumen の **フレームごとの全体更新パイプライン** のエントリポイントを担うファイル。  
Surface Cache の更新・Card キャプチャ・ライティング更新を統括し、  
`FDeferredShadingRenderer` から呼び出される。

---

## FDeferredShadingSceneRenderer::UpdateLumenScene

```cpp
void FDeferredShadingSceneRenderer::UpdateLumenScene(
    FRDGBuilder& GraphBuilder,
    FLumenSceneFrameTemporaries& FrameTemporaries,
    const FSceneViewFamily& ViewFamily);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `FrameTemporaries` | `FLumenSceneFrameTemporaries&` | フレームアトラス・バッファ参照を保持 |
| `ViewFamily` | `const FSceneViewFamily&` | ビューファミリ（解像度・品質設定）|

### 使用箇所
- [[ref_lumen_scene_rendering]] `FDeferredShadingRenderer::RenderLumenScene()` — フレーム冒頭

### 内部処理フロー

1. **凍結チェック（Freeze / Reset）**
   ```cpp
   if (CVarLumenSceneSurfaceCacheFreeze.GetValueOnRenderThread()) {
       return; // Surface Cache 更新をスキップ
   }
   if (CVarLumenSceneSurfaceCacheReset.GetValueOnRenderThread()) {
       LumenData.Reset(); // 全 Card / MeshCards をリセット
   }
   ```

2. **CPU 側プリミティブ更新**
   ```cpp
   // GPUDrivenUpdate のリードバックを処理（AddOps / RemoveOps）
   const FLumenSceneReadback::FBuffersRHI* ReadbackBuffers =
       LumenData.SceneReadback.GetLatestReadbackBuffers();
   if (ReadbackBuffers) {
       ProcessAddOps(ReadbackBuffers->AddOps, ...);
       ProcessRemoveOps(ReadbackBuffers->RemoveOps, ...);
   }
   
   // トランスフォーム変化の検出 → MeshCards 更新
   UpdateLumenScenePrimitives(Scene, FrameTemporaries);
   ```

3. **Surface Cache アロケーション（解像度決定・ページ割り当て）**
   ```cpp
   // カメラ距離 → DesiredLockedResLevel を計算
   // アトラス空き容量 → 実際の ResLevel を決定
   UpdateSurfaceCacheAllocationState(LumenData, Views, GPUMask, SurfaceCacheRequests);
   ```

4. **GPU バッファへのアップロード**
   ```cpp
   // Card / MeshCards / PrimitiveGroups が変化したものを GPU バッファへ転送
   UpdateLumenSceneBuffers(GraphBuilder, FrameTemporaries);
   ```

5. **GPU Driven Update**
   ```cpp
   // GPU コンピュートで可視性判定 → AddOps / RemoveOps 書き込み
   LumenScene::GPUDrivenUpdate(GraphBuilder, Scene, Views, FrameTemporaries);
   ```

6. **Card キャプチャ**
   ```cpp
   // キャプチャ対象 Card ページを優先度順に選定
   BuildCardUpdateList(LumenData, Views, SurfaceCacheRequests, LumenCardRenderer);
   
   // アトラス確保
   FCardCaptureAtlas CardCaptureAtlas =
       LumenScene::AllocateCardCaptureAtlas(GraphBuilder, CardPagesToRender);
   
   // 描画コマンドを積む
   LumenScene::AddCardCaptureDraws(Scene, ViewFamily, FrameTemporaries, CardPagesToRender, ...);
   
   // 実際のキャプチャ描画（非 Nanite / Nanite 分岐）
   RenderLumenCardCapture(GraphBuilder, CardCaptureAtlas, CardPagesToRender, ...);
   
   // Surface Cache アトラスへコピー
   UpdateLumenSurfaceCacheAtlas(GraphBuilder, FrameTemporaries, CardCaptureAtlas, ...);
   ```

7. **フィードバックバッファの送信**
   ```cpp
   LumenData.SurfaceCacheFeedback.SubmitFeedbackBuffer(View, GraphBuilder, FeedbackResources);
   ```

---

## FDeferredShadingSceneRenderer::RenderLumenSceneLighting

```cpp
void FDeferredShadingSceneRenderer::RenderLumenSceneLighting(
    FRDGBuilder& GraphBuilder,
    FLumenSceneFrameTemporaries& FrameTemporaries);
```

### 使用箇所
- [[ref_lumen_scene_rendering]] `RenderLumenScene()` — `UpdateLumenScene()` の後

### 内部処理フロー

```cpp
// 1. Radiosity フレームデータ初期化
LumenRadiosity::InitFrameTemporaries(GraphBuilder, FrameTemporaries, View, RadiosityTemporaries);

// 2. Direct Lighting 用更新コンテキスト構築
FLumenCardUpdateContext DirectLightingCardUpdateContext;
Lumen::BuildCardUpdateContext(GraphBuilder, ..., GLumenDirectLightingUpdateFactor,
    DirectLightingCardUpdateContext);

// 3. ページ → タイルに細分化
FLumenCardTileUpdateContext CardTileUpdateContext;
Lumen::SpliceCardPagesIntoTiles(GraphBuilder, DirectLightingCardUpdateContext,
    CardTileUpdateContext);

// 4. Direct Lighting の計算
if (IsEnabled(DirectLighting)) {
    RenderDirectLighting(GraphBuilder, DirectLightingCardUpdateContext, CardTileUpdateContext, ...);
}

// 5. Radiosity 用更新コンテキスト構築
FLumenCardUpdateContext RadiosityCardUpdateContext;
Lumen::BuildCardUpdateContext(GraphBuilder, ..., GLumenRadiosityUpdateFactor,
    RadiosityCardUpdateContext);

// 6. Radiosity の計算
if (LumenRadiosity::IsEnabled(ViewFamily)) {
    LumenRadiosity::AddRadiosityPass(GraphBuilder, RadiosityCardUpdateContext, RadiosityTemporaries, ...);
}

// 7. Final Lighting Atlas に合成
Lumen::CombineLumenSceneLighting(GraphBuilder, View, FrameTemporaries);
```

---

## 主要 CVar

### 更新制御

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.FastCameraMode` | 0 | 高速カメラ移動時に品質を下げて更新を高速化 |
| `r.LumenScene.ParallelUpdate` | 1 | CPU 側の更新をマルチスレッドで並列実行 |
| `r.LumenScene.PrimitivesPerTask` | 128 | タスクあたりのプリミティブ処理数 |
| `r.LumenScene.MeshCardsPerTask` | 128 | タスクあたりの MeshCards 処理数 |

### Surface Cache リセット・フリーズ

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.SurfaceCache.Freeze` | 0 | Surface Cache の更新を凍結（デバッグ用）|
| `r.LumenScene.SurfaceCache.FreezeUpdateFrame` | 0 | 同じサブセットを繰り返し更新（プロファイル用）|
| `r.LumenScene.SurfaceCache.Reset` | 0 | 全アトラスと Card を即時リセット |
| `r.LumenScene.SurfaceCache.ResetEveryNthFrame` | 0 | N フレームごとに自動リセット（0=無効）|

### Card キャプチャ

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.SurfaceCache.CardCapturesPerFrame` | 300 | 1 フレームあたりの最大キャプチャ Card 数 |
| `r.LumenScene.SurfaceCache.CardCaptureFactor` | 64 | キャプチャテクセル数の上限係数 |
| `r.LumenScene.SurfaceCache.RemovesPerFrame` | 512 | 1 フレームあたりの最大 MeshCards 削除数 |
| `r.LumenScene.SurfaceCache.CardMaxTexelDensity` | 0.2 | Card の最大テクセル密度（テクセル/cm）|
| `r.LumenScene.SurfaceCache.CardMaxResolution` | 512 | Card の最大解像度（px）|
| `r.LumenScene.SurfaceCache.NumFramesToKeepUnusedPages` | 256 | 未使用ページを保持するフレーム数 |
| `r.LumenScene.SurfaceCache.ResampleLighting` | 1 | Card 移動時に前フレーム照明をリサンプルするか |
| `r.LumenScene.SurfaceCache.Nanite.MultiView` | 1 | Nanite MultiView キャプチャを使うか |

---

## フレーム更新の全体フロー

```
FDeferredShadingRenderer::RenderLumenScene(GraphBuilder)
  │
  ├─ [CPU] UpdateLumenScene()
  │   ├─ Freeze / Reset チェック
  │   ├─ GPUDrivenUpdate リードバック処理（AddOps / RemoveOps のフラッシュ）
  │   ├─ UpdateLumenScenePrimitives() — トランスフォーム変化検出・MeshCards 更新
  │   ├─ UpdateSurfaceCacheAllocationState() — 解像度決定・ページ割り当て
  │   ├─ UpdateLumenSceneBuffers() — GPU バッファへのアップロード
  │   ├─ LumenScene::GPUDrivenUpdate() — GPU コンピュートで可視性判定
  │   └─ Card キャプチャ（AllocateCardCaptureAtlas → AddCardCaptureDraws → RenderLumenCardCapture）
  │
  ├─ [GPU] RenderLumenSceneLighting()
  │   ├─ LumenRadiosity::InitFrameTemporaries() — プローブアトラス初期化
  │   ├─ BuildCardUpdateContext(DirectLighting) — 更新対象ページのリストアップ
  │   ├─ SpliceCardPagesIntoTiles() — ページ→タイル細分化
  │   ├─ RenderDirectLighting() — SDF or HW RT シャドウ + Direct Lighting Atlas
  │   ├─ BuildCardUpdateContext(Radiosity) — Radiosity 用ページリストアップ
  │   ├─ LumenRadiosity::AddRadiosityPass() — プローブトレース → Indirect Lighting Atlas
  │   └─ CombineLumenSceneLighting() — Final Color Atlas に合成
  │
  └─ FLumenSceneFrameTemporaries 内のアトラス RDG テクスチャを後続パスに渡す
```

---

## FastCameraMode の効果

```cpp
// r.LumenScene.FastCameraMode = 1 にすると:
// - CardCapturesPerFrame が増加（より多くの Card を毎フレーム更新）
// - UpdateFactor が小さくなる（より頻繁にライティング更新）
// - 品質は低下するが、高速カメラ移動時のラグを抑制
//
// 実装的には UpdateFactor が半減し、CardCapturesPerFrame が 2 倍になる
// r.LumenScene.DirectLighting.UpdateFactor = 16 (通常 32)
// r.LumenScene.Radiosity.UpdateFactor = 32 (通常 64)
```

---

## ParallelUpdate の仕組み

```
r.LumenScene.ParallelUpdate = 1 の場合:
  CPU 側の MeshCards 更新 / プリミティブ更新を TaskGraph で並列化
  PrimitivesPerTask = 128 → 128 プリミティブを1タスクで処理
  MeshCardsPerTask  = 128 → 128 MeshCards を1タスクで処理

実装:
  ParallelForWithTaskContext(NumTasks, [](FTaskContext& Ctx, int32 TaskIndex) {
      // 担当プリミティブの MeshCards を更新
      for (int32 i = Start; i < End; i++) {
          UpdateMeshCards(i);
      }
  });
```

---

## UpdateLumenSurfaceCacheAtlas の役割

```
UpdateLumenSurfaceCacheAtlas(GraphBuilder, FrameTemporaries, CardCaptureAtlas, ...)
  │
  ├─ FClearLumenCardsPS — 新規割り当て Card のアトラス領域をクリア
  │     (FRasterizeToCardsVS + FClearLumenCardsPS でスキャッター描画)
  │
  ├─ FCopyCardCaptureLightingToAtlasPS — 照明リサンプルコピー
  │   ├─ bResampleLastLighting = true  → FResampledCardCaptureAtlas から照明をリサンプル
  │   └─ bResampleLastLighting = false → 0 初期化（初期キャプチャや変形後）
  │
  └─ GPU → Surface Cache アトラス への Albedo/Normal/Emissive コピー
       └─ GetSurfaceCacheCompression() に応じて:
            - BC7/BC5/BC6H 圧縮 (UAVAliasing 対応 GPU)
            - 非圧縮 (PF_R8G8B8A8, PF_R8G8, PF_FloatR11G11B10)
```

---

## ファイルの依存関係（インクルード）

```
LumenSceneRendering.cpp が含む主要ヘッダ:
  Lumen.h                     ← Lumen 基本定義
  LumenMeshCards.h            ← FLumenMeshCards
  LumenSurfaceCacheFeedback.h ← フィードバックバッファ
  LumenSceneLighting.h        ← FLumenCardUpdateContext
  LumenSceneCardCapture.h     ← FCardPageRenderData
  LumenTracingUtils.h         ← FLumenCardTracingParameters
  LumenRadiosity.h            ← LumenRadiosity::FFrameTemporaries
  LumenReflections.h          ← Reflections との連携
  Nanite/Nanite.h             ← Nanite キャプチャ
```
