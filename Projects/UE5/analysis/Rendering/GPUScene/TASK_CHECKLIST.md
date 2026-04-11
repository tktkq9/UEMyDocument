# GPUScene リファレンスドキュメント作成 タスクチェックリスト

作成目標: `Reference/` フォルダ以下に各ソースファイルのクラス・関数リファレンスを作成し、
対応する `Details/` 記事にリンクを追加する。

---

## フォルダ構成（完成形）

```
Rendering/GPUScene/
├── 09_gpuscene_overview.md
├── Details/
│   ├── a_gpuscene_buffers.md    ← リンク追加済み？ [x]
│   ├── b_gpuscene_update.md     ← リンク追加済み？ [x]
│   └── c_gpuscene_dynamic.md    ← リンク追加済み？ [x]
├── Reference/
│   ├── ref_gpuscene_class.md
│   └── ref_gpuscene_params.md
└── TASK_CHECKLIST.md（このファイル）
```

---

## Overview

| # | ファイル名 | 内容 | 状態 |
|---|-----------|------|------|
| OV-1 | `09_gpuscene_overview.md` | フレーム全体フロー・アーキテクチャ | [x] 完了 |

---

## Details — 3本

| # | ファイル名 | 内容 | 状態 |
|---|-----------|------|------|
| A-1 | `Details/a_gpuscene_buffers.md` | GPU バッファ構造・FSpanAllocator | [x] 完了 |
| A-2 | `Details/b_gpuscene_update.md` | BeginRender/Update/EndRender フロー | [x] 完了 |
| A-3 | `Details/c_gpuscene_dynamic.md` | 動的プリミティブ・FGPUScenePrimitiveCollector | [x] 完了 |

---

## Reference — 2本

| # | ファイル名 | 対象 | 状態 |
|---|-----------|------|------|
| R-1 | `Reference/ref_gpuscene_class.md` | FGPUScene クラス全体 | [x] 完了 |
| R-2 | `Reference/ref_gpuscene_params.md` | FGPUSceneResourceParameters / シェーダーバインド | [x] 完了 |

---

## 進捗サマリ

| グループ | 合計 | 完了 | 残り |
|---------|------|------|------|
| Overview | 1 | 1 | 0 |
| Details | 3 | 3 | 0 |
| Reference | 2 | 2 | 0 |
| **合計** | **6** | **6** | **0** |

---

## Phase 3 — コード実行フロー追加・Reference 強化

### Overview
- [x] 09_gpuscene_overview: BeginRender → Update/UpdateInternal → UploadDynamicPrimitives → EndRender 全体フロー追加

### Details（`## コード実行フロー` 追加）
- [x] a_gpuscene_buffers: AllocateInstanceSceneDataSlots → FSpanAllocator → ScatterUpload フロー
- [x] b_gpuscene_update: FGPUSceneScopeBeginEndHelper → Update → UpdateInternal → EndRender フロー
- [x] c_gpuscene_dynamic: FGPUScenePrimitiveCollector::Add → Commit → UploadDynamicPrimitiveShaderDataForView フロー

### Reference（メンバ変数テーブル化・`> [!note]-` 追加）
- [x] ref_gpuscene_class: FGPUScene メンバ変数テーブル・主要関数テーブル・3 callout
- [x] ref_gpuscene_params: FGPUSceneResourceParameters テーブル・SceneUB バインドフロー・3 callout

---

## 作業ルール

1. 1セッションで **同グループをまとめて**依頼するのが効率的
2. リファレンス記事には以下を含める：
   - クラス一覧（継承・役割の一言説明）
   - 主要関数一覧（引数・戻り値・役割）
   - 主要マクロ（`SHADER_PARAMETER`系）
   - 関連ファイルリンク
3. Detail記事の末尾に `## 関連リファレンス` セクションを追加してリンクを張る
4. 完了したタスクは `[x]` に更新してコミット
