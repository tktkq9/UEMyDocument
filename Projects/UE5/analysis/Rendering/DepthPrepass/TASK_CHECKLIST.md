# Depth Pre-pass / Velocity ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\`  
**GPU 対応**: [[GPU/DepthPrepass/TASK_CHECKLIST]]

---

## 概要ファイル

- [x] `13_depthprepass_overview.md` — Depth Pre-pass + Velocity Buffer の全体フロー・アーキテクチャ

---

## Phase 1: Details（2本）

- [x] `Details/a_depth_rendering.md`   — `RenderPrePass()` 全体フロー・Depth Only / Masked 分岐
- [x] `Details/b_velocity_rendering.md` — `RenderVelocities()` フロー・Skinned / Static / Nanite Velocity

---

## Phase 2: Reference（2本）

- [x] `Reference/ref_depth_rendering.md`   — `TDepthOnlyVS` / `FDepthOnlyPS` / `FDepthPassMeshProcessor`
- [x] `Reference/ref_velocity_rendering.md` — `FVelocityVS` / `FVelocityPS` / `FVelocityMeshProcessor` / `FVelocityPassUniformParameters`

---

## Phase 3: コード実行フロー追加

- [x] `13_depthprepass_overview.md`        — `RenderPrepassAndVelocity()` → PrePass → Nanite Depth → Velocity フロー追加
- [x] `Details/a_depth_rendering.md`       — `RenderPrePass()` 内部（Opaque → Masked → Early Z）フロー
- [x] `Details/b_velocity_rendering.md`    — `RenderVelocities()` → 各 MeshPassProcessor ディスパッチ フロー

---

## 進捗サマリ

| フェーズ | 合計 | 完了 | 残り |
|---------|------|------|------|
| 概要 | 1 | 1 | 0 |
| Phase 1: Details | 2 | 2 | 0 |
| Phase 2: Reference | 2 | 2 | 0 |
| Phase 3: Flow | 3 | 3 | 0 |
| **合計** | **8** | **8** | **0** |

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `DepthRendering.h/.cpp` | Depth Pre-pass パスの定義・`RenderPrePass()` |
| `VelocityRendering.h/.cpp` | Velocity Buffer パスの定義・`RenderVelocities()` |
| `MeshPassProcessor.h` | 共通 MeshPassProcessor 基底 |
