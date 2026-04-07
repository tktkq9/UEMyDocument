# リファレンス：LumenScene.cpp

- グループ: a - Surface Cache
- 上位: [[a_lumen_surface_cache]]
- 関連: [[ref_lumen_scene_data]] | [[ref_lumen_mesh_cards]] | [[ref_lumen_scene_gpu_driven_update]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenScene.cpp`

---

## 概要

Lumen シーンの**プリミティブ追加・削除・更新のCPU側オーケストレーション**を担うファイル。  
`FScene` 内の `FLumenSceneData` を毎フレーム最新状態に保ち、  
GPU バッファ（CardBuffer・MeshCardsBuffer など）へのアップロードを管理する。

---

## 主要 CVar（LumenScene.cpp で定義）

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.LumenScene.GlobalSDF.Resolution` | 252 | Global SDF の解像度（Lumen 有効時）|
| `r.LumenScene.GlobalSDF.ClipmapExtent` | 2500.0 | 最初のクリップマップの半径（cm）|
| `r.LumenScene.FarField` | 0 | 遠距離フィールド有効/無効（変更時プロキシ再作成）|
| `r.LumenScene.FarField.OcclusionOnly` | 0 | 遠距離をオクルージョンのみにして高速化 |
| `r.LumenScene.FarField.MaxTraceDistance` | 1.0e6 | 遠距離の最大トレース距離（cm）|
| `r.LumenScene.FarField.FarFieldDitherScale` | 200.0 | 近距離/遠距離の境界ディザ幅（cm）|
| `r.LumenScene.SurfaceCache.CardCapturesPerFrame` | 300 | 1 フレームあたり最大キャプチャ Card 数 |
| `r.LumenScene.SurfaceCache.CardCaptureFactor` | 64 | キャプチャテクセル上限係数 |
| `r.LumenScene.SurfaceCache.CardCaptureRefreshFraction` | 0.125 | 既存ページ再キャプチャ割合 |
| `r.LumenScene.SurfaceCache.Freeze` | 0 | 更新凍結（デバッグ用）|
| `r.LumenScene.SurfaceCache.Reset` | 0 | 全リセット（デバッグ用）|
| `r.LumenScene.SurfaceCache.AtlasSize` | 4096 | アトラスサイズ（px）|
| `r.LumenScene.ParallelUpdate` | 1 | 並列タスクで更新するか |
| `r.LumenScene.PrimitivesPerTask` | 128 | タスクあたりのプリミティブ数 |
| `r.LumenScene.MeshCardsPerTask` | 128 | タスクあたりの MeshCards 数 |
| `r.LumenScene.FastCameraMode` | 0 | 高速カメラ移動時の低品質モード |
| `r.LumenScene.UpdateViewOrigin` | 1 | View Origin の更新有効/無効 |

---

## 主要処理（関数レベルの概要）

### プリミティブ登録/削除

```
AddPrimitiveToLumenScene(FPrimitiveSceneInfo*)
  → FLumenPrimitiveGroup を作成
  → FLumenMeshCards を生成（FMeshCardsBuildData から Card を生成）
  → TSparseSpanArray<FLumenCard> に Card を追加
  → GPU バッファへのアップロードをスケジュール

RemovePrimitiveFromLumenScene(FPrimitiveSceneInfo*)
  → FLumenPrimitiveGroupRemoveInfo に遅延削除情報を記録
  → 次フレームのBegin時に実際に削除
```

### フレームごとの更新フロー

```
FDeferredShadingRenderer::UpdateLumenScene(GraphBuilder)
  │
  ├─ 遅延削除処理（RemoveOps の適用）
  ├─ トランスフォーム変化の検出 → MeshCards 更新
  ├─ Surface Cache アロケーション更新（解像度決定・ページ割り当て）
  ├─ GPU バッファへのアップロード（UpdateCardSceneBuffer）
  └─ FLumenSceneFrameTemporaries のアトラス RDG テクスチャ生成
```

### Lumen::UseFarField（実装例）

```cpp
// LumenScene.cpp
bool Lumen::UseFarField(const FSceneViewFamily& ViewFamily)
{
    return CVarLumenFarField.GetValueOnRenderThread() != 0
        && DoesPlatformSupportLumenGI(ViewFamily.GetShaderPlatform());
}
```
