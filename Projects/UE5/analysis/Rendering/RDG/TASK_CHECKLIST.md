# RDG リファレンスドキュメント作成 タスクチェックリスト

## フォルダ構成（完成形）

```
analysis/Rendering/RDG/
├── 10_rdg_overview.md
├── TASK_CHECKLIST.md
├── TASK_CHECKLIST_REF_ENHANCE.md
├── TASK_CHECKLIST_FLOW.md
├── Details/
│   ├── a_rdg_builder.md
│   ├── b_rdg_resources.md
│   ├── c_rdg_pass.md
│   ├── d_rdg_parameters.md
│   └── e_rdg_debug_trace.md
└── Reference/
    ├── ref_rdg_builder.md         (グループ a)
    ├── ref_rdg_definitions.md     (グループ a)
    ├── ref_rdg_resources.md       (グループ b)
    ├── ref_rdg_texture_subresource.md (グループ b)
    ├── ref_rdg_allocator.md       (グループ b)
    ├── ref_rdg_pass.md            (グループ c)
    ├── ref_rdg_private.md         (グループ c)
    ├── ref_rdg_parameter.md       (グループ d)
    ├── ref_rdg_blackboard.md      (グループ d)
    ├── ref_rdg_utils.md           (グループ d)
    ├── ref_rdg_event.md           (グループ d)
    ├── ref_rdg_validation.md      (グループ e)
    └── ref_rdg_trace.md           (グループ e)
```

> **注意**: Reference ファイルは Obsidian wikilink の解決性を保つため  
> `Reference/` 直下にフラット配置する。グループは各ファイルの frontmatter で管理する。

---

## Details（5件）

| # | ファイル名 | 対象サブシステム | 状態 |
|---|-----------|----------------|------|
| 1 | `Details/a_rdg_builder.md` | FRDGBuilder・グラフ構築フロー | [x] 完了 |
| 2 | `Details/b_rdg_resources.md` | リソース型（Texture / Buffer / View） | [x] 完了 |
| 3 | `Details/c_rdg_pass.md` | パス実行モデル・フラグ・並列化 | [x] 完了 |
| 4 | `Details/d_rdg_parameters.md` | シェーダーパラメータ・Blackboard・マクロ | [x] 完了 |
| 5 | `Details/e_rdg_debug_trace.md` | デバッグ・バリデーション・トレース | [x] 完了 |

---

## Reference（13件）

### グループ a: Builder

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| a-1 | `Reference/ref_rdg_builder.md` | `RenderGraphBuilder.h/.inl/.cpp` | [x] 完了 |
| a-2 | `Reference/ref_rdg_definitions.md` | `RenderGraphDefinitions.h`, `RenderGraphFwd.h` | [x] 完了 |

### グループ b: Resources

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| b-1 | `Reference/ref_rdg_resources.md` | `RenderGraphResources.h/.inl/.cpp` | [x] 完了 |
| b-2 | `Reference/ref_rdg_texture_subresource.md` | `RenderGraphTextureSubresource.h` | [x] 完了 |
| b-3 | `Reference/ref_rdg_allocator.md` | `RenderGraphAllocator.h/.cpp`, `RenderGraphResourcePool.cpp` | [x] 完了 |

### グループ c: Pass

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| c-1 | `Reference/ref_rdg_pass.md` | `RenderGraphPass.h/.cpp` | [x] 完了 |
| c-2 | `Reference/ref_rdg_private.md` | `RenderGraphPrivate.cpp` | [x] 完了 |

### グループ d: Parameters

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| d-1 | `Reference/ref_rdg_parameter.md` | `RenderGraphParameter.h`, `RenderGraphParameters.inl` | [x] 完了 |
| d-2 | `Reference/ref_rdg_blackboard.md` | `RenderGraphBlackboard.h/.cpp` | [x] 完了 |
| d-3 | `Reference/ref_rdg_utils.md` | `RenderGraphUtils.h/.cpp` | [x] 完了 |
| d-4 | `Reference/ref_rdg_event.md` | `RenderGraphEvent.h/.inl/.cpp` | [x] 完了 |

### グループ e: Debug

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| e-1 | `Reference/ref_rdg_validation.md` | `RenderGraphValidation.h/.cpp` | [x] 完了 |
| e-2 | `Reference/ref_rdg_trace.md` | `RenderGraphTrace.h/.cpp` | [x] 完了 |

---

## 進捗サマリ

| グループ | 合計 | 完了 | 残り |
|---------|------|------|------|
| Details | 5 | 5 | 0 |
| a: Builder | 2 | 2 | 0 |
| b: Resources | 3 | 3 | 0 |
| c: Pass | 2 | 2 | 0 |
| d: Parameters | 4 | 4 | 0 |
| e: Debug | 2 | 2 | 0 |
| **合計** | **18** | **18** | **0** |

---

## フェーズ管理

| フェーズ | チェックリスト | 状態 |
|---------|--------------|------|
| Phase 1: 初期作成 | `TASK_CHECKLIST.md` | [x] 完了 |
| Phase 2: Reference 拡張 | `TASK_CHECKLIST_REF_ENHANCE.md` | [ ] 未着手 |
| Phase 3: コード実行フロー | `TASK_CHECKLIST_FLOW.md` | [ ] 未着手 |

---

## 作業ルール

1. 1セッションで同グループをまとめて依頼するのが効率的
2. リファレンス記事には以下を含める：
   - クラス一覧（継承・役割）
   - 主要関数一覧（引数・戻り値・役割）
   - 主要マクロ（SHADER_PARAMETER 系）
   - 関連ファイルリンク
3. Details 記事の末尾に `## 関連リファレンス` セクションを追加
4. 完了したタスクは `[x]` に更新してコミット
