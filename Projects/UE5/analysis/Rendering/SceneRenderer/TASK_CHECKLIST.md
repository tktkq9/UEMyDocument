# SceneRenderer ドキュメントタスクリスト

**方針**: `01_rendering_overview.md` 階層に対して RDG と同じ3フェーズを適用する  
**フォーマット**: `D:\Learning\Projects\UE5\analysis\Rendering\HOWTO_create_system_docs.md` 参照  
**作業日**: 2026-04-11  
**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\`

---

## Phase 1: 概要・Details（基本構造ドキュメント）

### 概要ファイル（1件）

- [x] `01_rendering_overview.md` — 既存。フロー追加対象

### Details ファイル（4件）

- [x] `Details/a_deferred_renderer.md` — FDeferredShadingSceneRenderer の構造・パイプライン状態
- [x] `Details/b_scene.md` — FScene（プリミティブ・ライト・サブシステム）
- [x] `Details/c_view.md` — FSceneRenderer 基底クラス・FViewInfo
- [x] `Details/d_visibility.md` — BeginInitViews / EndInitViews / HZB 可視性カリング

---

## Phase 2: Reference 拡充（メンバ変数・呼び出し元・内部動作）

### Reference ファイル（4件）

- [x] `Reference/ref_deferred_shading_renderer.md` — FDeferredShadingSceneRenderer 全メソッド・状態構造体
- [x] `Reference/ref_scene_renderer.md` — FSceneRenderer 基底クラス
- [x] `Reference/ref_scene.md` — FScene 主要メンバ
- [x] `Reference/ref_view_info.md` — FViewInfo・FSceneView

---

## Phase 3: コード実行フロー追加

- [x] `01_rendering_overview.md` — Render() 大局フロー（SceneRenderBuilder → FRDGBuilder まで）
- [x] `Details/a_deferred_renderer.md` — Render() 内部フロー（全パス呼び出し順序）
- [x] `Details/b_scene.md` — FScene のライフタイムと更新タイミング
- [x] `Details/c_view.md` — FViewInfo 構築〜可視性計算フロー
- [x] `Details/d_visibility.md` — BeginInitViews → CullPrimitives → ComputeRelevance フロー

---

## 進捗サマリ

| フェーズ | 合計 | 完了 | 残り |
|---------|------|------|------|
| Phase 1: Details | 4 | 4 | 0 |
| Phase 2: Reference | 4 | 4 | 0 |
| Phase 3: Flow | 5 | 5 | 0 |
| **合計** | **13** | **13** | **0** |

---

## 主要ソースファイル

| ファイル | 行数 | 内容 |
|---------|------|------|
| `DeferredShadingRenderer.h` | 1180 | FDeferredShadingSceneRenderer クラス定義 |
| `DeferredShadingRenderer.cpp` | 4159 | Render() メイン実装 |
| `SceneRendering.h` | 3137 | FSceneRenderer / FViewInfo |
| `ScenePrivate.h` | 2874 | FScene 本体 |
| `SceneVisibility.cpp` | — | BeginInitViews / CullPrimitives |
| `BasePassRendering.cpp` | — | RenderBasePass |
| `LightRendering.cpp` | — | RenderLights |
