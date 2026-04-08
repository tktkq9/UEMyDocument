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
  │   ├─ 遅延削除の適用（RemoveOps のフラッシュ）
  │   ├─ GPU Driven Update のリードバック処理
  │   ├─ トランスフォーム変化の検出 → MeshCards 更新
  │   ├─ Surface Cache アロケーション（解像度決定・ページ割り当て）
  │   └─ GPU バッファへのアップロード（Card / MeshCards / PrimitiveGroups）
  │
  ├─ [GPU] LumenScene::GPUDrivenUpdate()
  │   └─ GPU コンピュートで可視性判定 → AddOps / RemoveOps 書き込み
  │
  ├─ [GPU] Card キャプチャ（AllocateCardCaptureAtlas → AddCardCaptureDraws）
  │   ├─ Freeze / Reset チェック
  │   ├─ キャプチャ対象 Card ページを優先度順に選定
  │   └─ Nanite / 非 Nanite メッシュを描画 → Albedo / Normal / Emissive / Depth
  │
  ├─ [GPU] RenderLumenSceneLighting()
  │   ├─ BuildCardUpdateContext()    ← 更新対象ページのリストアップ
  │   ├─ RenderDirectLighting()     ← Direct Lighting 計算（SDF or HW RT）
  │   ├─ RenderRadiosity()          ← Indirect Lighting 計算（プローブトレース）
  │   └─ CombineLumenSceneLighting() ← Final Color アトラスに合成
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
```

---

## ParallelUpdate の仕組み

```
r.LumenScene.ParallelUpdate = 1 の場合:
  CPU 側の MeshCards 更新 / プリミティブ更新を TaskGraph で並列化
  PrimitivesPerTask = 128 → 128 プリミティブを1タスクで処理
  MeshCardsPerTask  = 128 → 128 MeshCards を1タスクで処理
  → 大規模シーンで CPU ボトルネックを軽減
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
