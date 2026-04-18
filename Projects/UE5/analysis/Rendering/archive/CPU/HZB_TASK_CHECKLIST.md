# HZB（Hierarchical Z-Buffer）ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\`  
**GPU 対応**: [[GPU/HZB/TASK_CHECKLIST]]

---

## 概要ファイル

- [x] `21_hzb_overview.md` — HZB の全体フロー・Occlusion Query との連携・使用箇所一覧

---

## Phase 1: Details（2本）

- [x] `Details/a_hzb_build.md`     — `BuildHZB()` フロー・SceneDepth からのミップチェーン生成・Min/Max Reduce
- [x] `Details/b_hzb_occlusion.md` — `FHZBOcclusionTester` フロー・クエリ発行・結果読み取り・フィードバック

---

## Phase 2: Reference（2本）

- [x] `Reference/ref_hzb_resources.md`  — `FHZBBuildParameters` / `FSceneHZB` クラス / HZB テクスチャ仕様
- [x] `Reference/ref_hzb_occlusion.md`  — `FHZBOcclusionTester` / `FHZBOcclusionQuery` / `HZBOcclusion.cpp` 関数群

---

## Phase 3: コード実行フロー追加

- [x] `21_hzb_overview.md`          — `BuildHZB()` → `SubmitHZB()` → `RenderOcclusion()` フロー追加
- [x] `Details/a_hzb_build.md`      — SceneDepth → Downsample → MipChain 生成フロー
- [x] `Details/b_hzb_occlusion.md`  — クエリ発行 → GPU 実行 → ReadbackBuffer → PrimitiveVisibilityMap 更新フロー

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
| `SceneHZB.h/.cpp` | `BuildHZB()` / HZB テクスチャ管理 |
| `HZBOcclusion.h/.cpp` | `FHZBOcclusionTester` / クエリ管理 |
