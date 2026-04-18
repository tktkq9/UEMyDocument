# Base Pass（GBuffer）ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\`  
**GPU 対応**: [[GPU/BasePass/TASK_CHECKLIST]]

---

## 概要ファイル

- [x] `14_basepass_overview.md` — Base Pass の全体フロー・GBuffer レイアウト・マテリアル生成

---

## Phase 1: Details（3本）

- [x] `Details/a_basepass_pipeline.md` — `RenderBasePass()` 全体フロー・非 Nanite / Nanite 分岐
- [x] `Details/b_gbuffer_layout.md`    — GBuffer A/B/C/D/E/F チャンネル定義・Substrate 時の差異
- [x] `Details/c_basepass_material.md` — マテリアル別シェーダー生成・`TBasePassMeshProcessor` の仕組み

---

## Phase 2: Reference（3本）

- [x] `Reference/ref_basepass_renderer.md` — `TBasePassMeshProcessor` / `FBasePassVS` / `FBasePassPS` クラス群
- [x] `Reference/ref_basepass_common.md`   — `FBasePassUniformParameters` / `BasePassCommon.ush` 定数群
- [x] `Reference/ref_gbuffer_textures.md`  — `FSceneTextures` / `FGBufferData` / チャンネル対応表

---

## Phase 3: コード実行フロー追加

- [x] `14_basepass_overview.md`        — `RenderBasePass()` → MeshPassProcessor → RDG Pass 発行 フロー追加
- [x] `Details/a_basepass_pipeline.md` — `SetupBasePassState()` → `DispatchDraw()` → Nanite GBuffer Resolve フロー
- [x] `Details/b_gbuffer_layout.md`    — GBuffer 各テクスチャの生成〜バインド〜デコードフロー
- [x] `Details/c_basepass_material.md` — マテリアルシェーダー排列 → `TBasePassMeshProcessor::AddMeshBatch()` フロー

---

## 進捗サマリ

| フェーズ | 合計 | 完了 | 残り |
|---------|------|------|------|
| 概要 | 1 | 1 | 0 |
| Phase 1: Details | 3 | 3 | 0 |
| Phase 2: Reference | 3 | 3 | 0 |
| Phase 3: Flow | 4 | 4 | 0 |
| **合計** | **11** | **11** | **0** |

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `BasePassRendering.h/.cpp` | Base Pass MeshPassProcessor・RenderBasePass() |
| `BasePassCommon.ush` | GBuffer 定数・共通マクロ |
| `SceneTextures.h/.cpp` | FSceneTextures（GBuffer テクスチャ束）|
| `GBufferInfo.h` | FGBufferData デコード構造体 |
