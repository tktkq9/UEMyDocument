# Translucency ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\`  
**GPU 対応**: [[GPU/Translucency/TASK_CHECKLIST]]

---

## 概要ファイル

- [x] `20_translucency_overview.md` — Translucency の全体フロー・パス分割（Before/After DOF）・OIT 概要

---

## Phase 1: Details（3本）

- [x] `Details/a_translucent_rendering.md` — `RenderTranslucency()` フロー・深度ソート・Before/After DOF 分割
- [x] `Details/b_translucent_lighting.md`  — TranslucencyVolume への照明注入・Lumen Translucency Radiance Cache との連携
- [x] `Details/c_oit.md`                   — Order-Independent Translucency（Weighted Blended OIT / Pixel Linked List）

---

## Phase 2: Reference（2本）

- [x] `Reference/ref_translucent_rendering.md` — `FTranslucencyPassResources` / `FTranslucentPrimSet` / `FSortedTriangle`
- [x] `Reference/ref_translucent_lighting.md`  — `FTranslucentSelfShadowParameters` / `FTranslucentLightingVolumeTextures`

---

## Phase 3: コード実行フロー追加

- [x] `20_translucency_overview.md`          — `RenderTranslucency()` → Separate RT → Standard RT → OIT Composite フロー追加
- [x] `Details/a_translucent_rendering.md`   — 深度ソート → パス分割 → 各メッシュ描画フロー
- [x] `Details/b_translucent_lighting.md`    — InjectTranslucencyVolume → FilterVolume → 半透明シェーダーでのサンプリングフロー
- [x] `Details/c_oit.md`                     — OITAccumulate → Composite パスフロー

---

## 進捗サマリ

| フェーズ | 合計 | 完了 | 残り |
|---------|------|------|------|
| 概要 | 1 | 1 | 0 |
| Phase 1: Details | 3 | 3 | 0 |
| Phase 2: Reference | 2 | 2 | 0 |
| Phase 3: Flow | 4 | 4 | 0 |
| **合計** | **10** | **10** | **0** |

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `TranslucentRendering.h/.cpp` | `RenderTranslucency()` メインパス |
| `TranslucentLightingShaders.h/.cpp` | 照明ボリューム注入・補間パス |
| `BasePassRendering.cpp` | 半透明マテリアルの BasePass（Forward Shading）|
| `OIT/OIT.h/.cpp` | Order-Independent Translucency（MLAB / SortedTriangles）|
