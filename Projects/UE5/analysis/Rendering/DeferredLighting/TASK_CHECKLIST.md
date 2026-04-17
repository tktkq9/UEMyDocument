# Deferred Lighting ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\`  
**GPU 対応**: [[GPU/DeferredLighting/TASK_CHECKLIST]]

---

## 概要ファイル

- [x] `16_deferred_lighting_overview.md` — Deferred Lighting の全体フロー・ライト種別・Clustered/Tiled 選択

---

## Phase 1: Details（4本）

- [x] `Details/a_light_rendering.md`     — `RenderLights()` → `RenderLight()` 全体フロー・ライト種別ディスパッチ
- [x] `Details/b_clustered_tiled.md`     — `ClusteredDeferredShading` / `TiledDeferredShading` の仕組み・使い分け
- [x] `Details/c_reflection_skylight.md` — `RenderDeferredReflectionsAndSkyLighting()` フロー・反射キャプチャ合成
- [x] `Details/d_light_functions.md`     — Light Function マテリアルのライトへの適用パス

---

## Phase 2: Reference（3本）

- [x] `Reference/ref_light_rendering.md`  — `FDeferredLightVS` / `FDeferredLightUniformStruct` / `ELightOcclusionType`
- [x] `Reference/ref_clustered_tiled.md`  — `FClusteredShadingPS` / `FClusteredShadingVS` / `EClusterPassInputType`
- [x] `Reference/ref_light_params.md`     — `FSortedLightSetSceneInfo` / `FLightSceneProxy` / `FLightSceneInfo`

---

## Phase 3: コード実行フロー追加

- [x] `16_deferred_lighting_overview.md` — `RenderLights()` → ライト分類 → Clustered/Direct → Shadow Projection フロー追加
- [x] `Details/a_light_rendering.md`     — ライト種別ごとの `RenderLight()` ディスパッチフロー
- [x] `Details/b_clustered_tiled.md`     — ライト割り当て → Cluster/Tile → CS ディスパッチフロー
- [x] `Details/c_reflection_skylight.md` — ReflectionCapture → SkyLight → Lumen Reflection 合成フロー
- [x] `Details/d_light_functions.md`     — LightFunctionMaterial → テクスチャ生成 → ライトへの適用フロー

---

## 進捗サマリ

| フェーズ | 合計 | 完了 | 残り |
|---------|------|------|------|
| 概要 | 1 | 1 | 0 |
| Phase 1: Details | 4 | 4 | 0 |
| Phase 2: Reference | 3 | 3 | 0 |
| Phase 3: Flow | 5 | 5 | 0 |
| **合計** | **13** | **13** | **0** |

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `LightRendering.h/.cpp` | `RenderLights()` メインオーケストレーション |
| `ClusteredDeferredShadingPass.cpp` | Clustered Deferred Shading |
| `DeferredShadingRenderer.cpp` | `RenderDeferredReflectionsAndSkyLighting()` 含む |
| `ReflectionEnvironmentCapture.h/.cpp` | ReflectionCapture 関連 |
| `LightFunctionRendering.cpp` | Light Function マテリアル適用 |
