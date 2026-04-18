# Substrate ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\Substrate\`  
**注意**: `r.Substrate=1` でのみ有効。旧 Shading Models は `r.Substrate=0` 時に使用。

---

## 概要ファイル

- [x] `08_substrate_overview.md` — アーキテクチャ・タイル分類・主要クラス・CVar・フロー

---

## Phase 1: Details（3本）

- [x] `Details/a_substrate_material.md`  — クロージャ・レイヤー構造・MaterialTextureArray への書き込み
- [x] `Details/b_substrate_classify.md`  — タイル分類パス（ステンシルビット・Fast/Single/Complex タイルリスト）
- [x] `Details/c_substrate_lighting.md`  — Substrate 対応ライティングパス・RoughRefraction・SubSurface

---

## Phase 2: Reference（3本）

- [x] `Reference/ref_substrate_data.md`     — `FSubstrateSceneData` / `FSubstrateViewData` 全メンバ
- [x] `Reference/ref_substrate_params.md`   — `FSubstrateGlobalUniformParameters` / シェーダーバインド
- [x] `Reference/ref_substrate_classify.md` — 分類パス関数群 / タイルリストバッファ / `FSubstrateTilePassVS`

---

## Phase 3: コード実行フロー追加

- [x] `08_substrate_overview.md`               — InitialiseSubstrateFrameSceneData → BasePass → Classify → Lighting フロー追加
- [x] `Details/a_substrate_material.md`        — マテリアルシェーダーが MaterialTextureArray に書き込むフロー
- [x] `Details/b_substrate_classify.md`        — AddSubstrateMaterialClassificationPass → タイルリスト生成 フロー
- [x] `Details/c_substrate_lighting.md`        — Indirect Dispatch によるタイル単位ライティング + RoughRefraction フロー

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
| `Substrate.h/.cpp` | フレームデータ初期化・タイル分類・各パスの追加 |
| `SubstrateRoughRefraction.cpp` | 不透明粗面屈折パス |
| `SubstrateVisualize.cpp` | デバッグビジュアライゼーション |
| `Glint/` | グリント（マイクロファセットきらめき）処理サブフォルダ |
