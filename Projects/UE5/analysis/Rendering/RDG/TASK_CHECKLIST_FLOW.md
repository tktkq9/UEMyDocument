# RDG コード実行フロー 追記タスクリスト

**方針**: 概要・Details 各ファイルに「どこから処理が始まり、どの関数・クラスが呼ばれるか」を追記する  
**追記形式**: `## コード実行フロー` セクション（エントリポイント → 段階的コールチェーン + コードスニペット）  
**フォーマット**: `D:\Learning\Projects\UE5\analysis\Rendering\HOWTO_create_system_docs.md`（セクション8）参照  
**作業開始日**: 未着手  
**優先度**: 概要 → a → b → c → d → e の順

---

## タスク一覧

### 概要ファイル（1件）

- [ ] `10_rdg_overview.md` — `FDeferredShadingSceneRenderer::Render()` → `FRDGBuilder` 生成・`Execute()` までの大局フロー

**追記内容**:
- `BeginRenderingViewFamily()` → `FRDGBuilder GraphBuilder(...)` → 各レンダリングパスへの呼び出し
- `GraphBuilder.Execute()` が実行されるタイミング
- フレーム内での RDG グラフの生成・実行の全体像

---

### a: Builder（1件）

- [ ] `Details/a_rdg_builder.md` — `FRDGBuilder` 構築〜`Execute()` の内部フロー

**追記内容**:
- `Execute()` 内の `Compile()` → バリア挿入 → `ExecutePasses()` の連鎖
- `SetupPassDependencies()` → `CullPasses()` → `CollectPassBarriers()` → `MergeRenderPasses()` → `ExecutePasses()`
- 並列実行（`ParallelExecute`）時のタスクグラフ生成

---

### b: Resources（1件）

- [ ] `Details/b_rdg_resources.md` — リソースライフタイムとアロケーションフロー

**追記内容**:
- `CreateTexture()` → グラフ内での遅延確保 → 最初の参照パス時の実 GPU メモリ確保
- `RegisterExternalTexture()` → 外部 `IPooledRenderTarget` の RDG 統合フロー
- `QueueTextureExtraction()` → `Execute()` 後の外部バッファへの書き戻し

---

### c: Pass（1件）

- [ ] `Details/c_rdg_pass.md` — パス追加〜実行・バリア発行のフロー

**追記内容**:
- `AddPass()` → `TRDGLambdaPass` 生成 → `FRDGPassRegistry` への登録
- `Compile()` フェーズでのパスカリング（`CullPasses()`）とバリア計算（`CollectPassBarriers()`）
- `ExecutePasses()` での RenderPass マージ・AsyncCompute フェンスの挿入タイミング

---

### d: Parameters（1件）

- [ ] `Details/d_rdg_parameters.md` — パラメータ構造体の解析・Blackboard のフロー

**追記内容**:
- `AddPass(Params, ...)` → `FRDGParameterStruct::EnumerateTextures/Buffers()` によるリソース依存解析
- `Blackboard.Create<T>()` / `GetChecked<T>()` の内部インデックス解決フロー
- `RDG_EVENT_NAME` / `RDG_EVENT_SCOPE` がプロファイラに渡るまでの流れ

---

### e: Debug（1件）

- [ ] `Details/e_rdg_debug_trace.md` — バリデーション・トレースの実行タイミング

**追記内容**:
- `FRDGUserValidation::ValidateAddPass()` が呼ばれるタイミングとスタック
- `FRDGTrace::AddResource()` / `AddTexturePassDependency()` の呼び出しポイント
- `r.RDG.ImmediateMode` が有効な場合のラムダ実行パスの違い

---

## 進捗サマリ

| ファイル | 状態 |
|---------|------|
| `10_rdg_overview.md` | [ ] 未着手 |
| `a_rdg_builder.md` | [ ] 未着手 |
| `b_rdg_resources.md` | [ ] 未着手 |
| `c_rdg_pass.md` | [ ] 未着手 |
| `d_rdg_parameters.md` | [ ] 未着手 |
| `e_rdg_debug_trace.md` | [ ] 未着手 |

**合計**: 6 ファイル  
**完了数**: 0 / 6

---

## Claude への依頼テンプレート

```
RDGのコード実行フロー追記をお願いします。

対象チェックリスト: D:\Learning\Projects\UE5\analysis\Rendering\RDG\TASK_CHECKLIST_FLOW.md
フォーマット: D:\Learning\Projects\UE5\analysis\Rendering\HOWTO_create_system_docs.md（セクション8）
ソースコード: D:\UnrealEngine\Engine\Source\Runtime\RenderCore\

{ファイル名}をお願いします。
```
