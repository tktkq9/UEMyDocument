# RDG Reference 拡張タスクリスト

**方針**: 各 Reference ファイルの関数ブロックに「パラメータ表 / 呼び出し元 / 内部動作」を追加  
**フォーマット**: `D:\Learning\Projects\UE5\analysis\Rendering\HOWTO_create_system_docs.md`（セクション7）参照  
**実例**: `D:\Learning\Projects\UE5\analysis\Rendering\Lumen\Reference\`  
**作業開始日**: 未着手  
**優先度**: グループ a → b → c → d → e の順

---

## グループ a: Builder（2件）

- [ ] `Reference/ref_rdg_builder.md`
- [ ] `Reference/ref_rdg_definitions.md`

## グループ b: Resources（3件）

- [ ] `Reference/ref_rdg_resources.md`
- [ ] `Reference/ref_rdg_texture_subresource.md`
- [ ] `Reference/ref_rdg_allocator.md`

## グループ c: Pass（2件）

- [ ] `Reference/ref_rdg_pass.md`
- [ ] `Reference/ref_rdg_private.md`

## グループ d: Parameters（4件）

- [ ] `Reference/ref_rdg_parameter.md`
- [ ] `Reference/ref_rdg_blackboard.md`
- [ ] `Reference/ref_rdg_utils.md`
- [ ] `Reference/ref_rdg_event.md`

## グループ e: Debug（2件）

- [ ] `Reference/ref_rdg_validation.md`
- [ ] `Reference/ref_rdg_trace.md`

---

## 追記要件（全ファイル共通）

1. **全関数を網羅** — 主要でない関数も含め `> [!note]-` 折りたたみで収録
2. **メンバ変数の説明** — 各クラスに `### メンバ変数` テーブルを追加
3. **使用箇所とリンク** — どのクラス・関数から使われるかを `[[ref_xxx]]` 付きで記載
4. **内部処理フロー** — 処理量の多い関数はステップごとにコード概要を添付

---

## 進捗サマリ

| グループ | 合計 | 完了 | 残り |
|---------|------|------|------|
| a: Builder | 2 | 0 | 2 |
| b: Resources | 3 | 0 | 3 |
| c: Pass | 2 | 0 | 2 |
| d: Parameters | 4 | 0 | 4 |
| e: Debug | 2 | 0 | 2 |
| **合計** | **13** | **0** | **13** |

---

## Claude への依頼テンプレート

```
RDGのReferenceドキュメントを拡張してください。

対象チェックリスト: D:\Learning\Projects\UE5\analysis\Rendering\RDG\TASK_CHECKLIST_REF_ENHANCE.md
フォーマット例: D:\Learning\Projects\UE5\analysis\Rendering\HOWTO_create_system_docs.md（セクション7）
Lumenの実例: D:\Learning\Projects\UE5\analysis\Rendering\Lumen\Reference\

グループ{a/b/c/d/e}をお願いします。
```
