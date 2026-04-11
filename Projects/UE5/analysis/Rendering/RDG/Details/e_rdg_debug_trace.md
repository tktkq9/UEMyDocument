# E: RDG デバッグ・バリデーション・トレース

- 対象: `RenderGraphValidation.h/.cpp`, `RenderGraphTrace.h/.cpp`, `RenderGraphPrivate.cpp`
- 上位: [[10_rdg_overview]]
- 関連: [[a_rdg_builder]] | [[c_rdg_pass]]

---

## 概要

RDG のデバッグ・検証・プロファイリング機能を提供するサブシステム群。  
すべてビルド構成マクロでコンパイルアウトされ、リリースビルドではゼロオーバーヘッド。

| 機能 | クラス/マクロ | 有効条件 |
|------|------------|---------|
| ユーザー API バリデーション | `FRDGUserValidation` | `RDG_ENABLE_DEBUG` |
| 即時実行モード | `r.RDG.ImmediateMode` | `RDG_ENABLE_DEBUG` |
| リソースのクラブ | `r.RDG.ClobberResources` | `RDG_ENABLE_DEBUG` |
| Insights トレース出力 | `FRDGTrace` | `RDG_ENABLE_TRACE` |

---

## バリデーション（FRDGUserValidation）

`FRDGBuilder` の内部に保持されるバリデーション層。  
ユーザー起因の API 誤用をセットアップ段階（`AddPass` 前後）で検出する。

```
AddPass 呼び出し時の検証フロー:
  AddPass(Name, Params, Flags, Lambda)
    │
    ├─ パラメータ構造体の全リソースがグラフに登録済みか確認
    ├─ UAV への書き込みパスが NeverCull なしで参照されているか確認
    └─ ラムダの引数型と ERDGPassFlags の整合性確認
```

### 主な検証エラーメッセージのパターン

| エラー | 原因 |
|-------|------|
| `"...not registered in the graph"` | `FRDGTextureRef` をグラフ外で取得したテクスチャに使った |
| `"duplicate Create called on struct"` | `Blackboard.Create<T>()` を同型で 2 回呼んだ |
| `"RDGBlackboard Get failed to find..."` | `GetChecked<T>()` で登録されていない型を取得しようとした |

---

## 即時実行モード（r.RDG.ImmediateMode）

通常、パスのラムダは `Execute()` まで遅延実行される。  
即時モードでは `AddPass` した瞬間にラムダが実行される。

```cpp
// 通常モード:  AddPass → ... → Execute() → ラムダ実行
// 即時モード:  AddPass → 即ラムダ実行（グラフコンパイルは省略）
```

**用途:** クラッシュ時のコールスタックを追う。  
即時モードではバリア最適化・マージ・カリングは行われない。

---

## デバッグ CVar 一覧

### 実行制御

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RDG.ImmediateMode` | 0 | パスを宣言と同時に即時実行 |
| `r.RDG.Validation` | 1 | API バリデーションの有効/無効 |
| `r.RDG.Debug.FlushGPU` | 0 | 各パス後に GPU フラッシュ |

### リソース制御

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RDG.ClobberResources` | 0 | 未初期化リソースをランダム値で埋める |
| `r.RDG.Debug.ExtendResourceLifetimes` | 0 | リソースライフタイムを延長してエイリアシングを無効化 |
| `r.RDG.Debug.DisableTransientResources` | 0 | Transient アロケータを使わない |

### ダンプ・可視化

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RDG.Debug.ResourceFilter` | `""` | フィルタ対象のリソース名 |
| `r.RDG.DumpGraph` | 0 | グラフ構造を Dot ファイルにダンプ |

---

## Unreal Insights トレース（FRDGTrace）

`RDG_ENABLE_TRACE` が有効なビルドで、Unreal Insights に RDG の詳細データを送信する。

```
FRDGBuilder コンストラクタ
  └─ FRDGTrace::OutputGraphBegin()     ← グラフ開始マーク

CreateTexture / CreateBuffer ごとに
  └─ FRDGTrace::AddResource()          ← リソース情報を送信

AddPass のパラメータ解析ごとに
  ├─ FRDGTrace::AddTexturePassDependency()  ← テクスチャ依存
  └─ FRDGTrace::AddBufferPassDependency()   ← バッファ依存

FRDGBuilder デストラクタ
  └─ FRDGTrace::OutputGraphEnd()       ← グラフ終了マーク + Transient 統計
```

### Insights での確認方法

1. Unreal Insights を起動し `RDGChannel` を接続設定で有効化
2. ゲーム実行中に Insights がデータを受信
3. **Render Graph** ビューでグラフの階層・リソースライフタイム・依存関係を確認

---

## 関連リファレンス

| リファレンス | 対象ソース |
|------------|----------|
| [[ref_rdg_validation]] | `RenderGraphValidation.h/.cpp` |
| [[ref_rdg_trace]] | `RenderGraphTrace.h/.cpp` |
| [[ref_rdg_private]] | `RenderGraphPrivate.cpp` |

---

## コード実行フロー

### エントリポイント

```
──── バリデーション呼び出しタイミング ────

FRDGBuilder::CreateTexture(Desc, Name, Flags)
  └─ IF_RDG_ENABLE_DEBUG
      → UserValidation.ValidateCreateTexture(Desc, Name, Flags)
         → Desc が有効か、Name が null でないかを検証

FRDGBuilder::RegisterExternalTexture(PooledRT, Name, Flags)
  └─ IF_RDG_ENABLE_DEBUG
      → UserValidation.ValidateRegisterExternalTexture(PooledRT, Name, Flags)
         → 同一リソースの二重登録を検出

FRDGBuilder::AddPass(Name, Params, Flags, Lambda)
  ├─ IF_RDG_ENABLE_DEBUG
  │   → UserValidation.ValidateAddPass(Params, Metadata, Name, Flags)  [パラメータ前検証]
  │      → Params 内の全 RDG リソースがグラフに登録済みか確認
  │      → UAV 書き込みが NeverCull なしで参照されているかを確認
  │      → AsyncCompute / Await フラグと Inline 引数の整合性確認
  └─ IF_RDG_ENABLE_DEBUG
      → UserValidation.ValidateAddPass(Pass)  [パス生成後の最終検証]

FRDGBuilder::Execute()
  ├─ IF_RDG_ENABLE_DEBUG → UserValidation.ValidateExecuteBegin()
  │
  ├─ [パスごとのループ]
  │   ├─ IF_RDG_ENABLE_DEBUG → UserValidation.ValidateExecutePassBegin(Pass)
  │   │   → SetAllowRHIAccess(Pass, true)  ← パスラムダ中のみ GetRHI() を許可
  │   ├─ [ラムダ実行]
  │   └─ IF_RDG_ENABLE_DEBUG → UserValidation.ValidateExecutePassEnd(Pass)
  │       → SetAllowRHIAccess(Pass, false) ← ラムダ外での GetRHI() を禁止に戻す
  │
  └─ IF_RDG_ENABLE_DEBUG → UserValidation.ValidateExecuteEnd()

──── トレース呼び出しタイミング（RDG_ENABLE_TRACE） ────

FRDGBuilder コンストラクタ
  └─ IF_RDG_ENABLE_TRACE → Trace.OutputGraphBegin()
      → GraphStartCycles を記録、bEnabled = RDGChannel.IsEnabled()

FRDGBuilder::CreateTexture() / CreateBuffer()
  └─ IF_RDG_ENABLE_TRACE → Trace.AddResource(Resource)
      → ResourceOrder をインクリメント
      → UE_TRACE: RDG_RESOURCE イベント（名前・サイズ・フォーマット）を送信

FRDGBuilder::SetupParameterPass()  ← AddPass から呼ばれる
  └─ IF_RDG_ENABLE_TRACE
      → Trace.AddTexturePassDependency(Texture, Pass)
      → Trace.AddBufferPassDependency(Buffer, Pass)
         → UE_TRACE: RDG_DEPENDENCY イベント（リソース → パスの有向グラフ）

FRDGBuilder::Execute() 完了
  └─ IF_RDG_ENABLE_TRACE → Trace.OutputGraphEnd(GraphBuilder)
      → GraphStartCycles からの経過時間を計算
      → TransientAllocationStats を収集
      → UE_TRACE: RDG_GRAPH_END イベントを送信

──── ImmediateMode が有効な場合の特殊フロー ────

r.RDG.ImmediateMode = 1 の場合:
  AddPass(Name, Params, Flags, Lambda)
    → 通常: ラムダを登録して defer
    → ImmediateMode: 登録と同時にラムダを即実行
       バリア計算・カリング・マージは省略
       → クラッシュ時のコールスタックが AddPass 地点を指す
```

### フロー詳細

1. **ValidateAddPass() のスタック**
   ```
   FRDGBuilder::AddPass(Name, Params, Flags, Lambda)
     │
     └─ [RDG_ENABLE_DEBUG]
         FRDGUserValidation::ValidateAddPass(Params, Metadata, Name, Flags)
           ├─ FRDGParameterStruct::EnumerateTextures で全テクスチャを走査
           │   → ResourceMap.Contains(Texture) == false → ensureMsgf で警告
           ├─ EnumerateBuffers で全バッファを走査（同様）
           └─ Flags の整合性確認（AsyncCompute + Inline 引数 → エラー）
   ```

2. **SetAllowRHIAccess — ラムダ外での GetRHI() 禁止**
   ```cpp
   // ExecutePassBegin 前後で切り替わる（静的メソッド）
   FRDGUserValidation::SetAllowRHIAccess(Pass, true);
   // ← この区間のみ FRDGTexture::GetRHI() が呼べる
   Pass->Execute(RHICmdList);  // ユーザーラムダ実行
   FRDGUserValidation::SetAllowRHIAccess(Pass, false);
   // ← ここ以降で GetRHI() を呼ぶと ensure でエラー
   ```
   - `GRDGAllowRHIAccess` フラグが `FRDGTexture::GetRHI()` 内でチェックされる
   - ラムダ外での `GetRHI()` 呼び出しをビルド時ではなく実行時に検出できる

3. **FRDGTrace::AddResource() — CreateTexture 直後**
   ```cpp
   // FRDGBuilder::CreateTexture の末尾（擬似コード）
   FRDGTexture* Texture = Allocator.Alloc<FRDGTexture>(Desc, Name, Flags);
   IF_RDG_ENABLE_TRACE(Trace.AddResource(Texture));
   // → bEnabled チェック → UE_TRACE_LOG(RDGChannel, RDGResource, ...)
   //    ResourceOrder++ でタイムライン上の順序情報を付与
   ```

4. **ImmediateMode での実行パス比較**

   | フェーズ | 通常モード | ImmediateMode |
   |---------|-----------|---------------|
   | AddPass | ラムダを登録のみ | ラムダを即実行 |
   | Compile | Execute() 内で実施 | スキップ |
   | バリア最適化 | Split Barrier 使用 | 即時バリア発行 |
   | カリング | CullPasses() で実施 | スキップ（全パス実行） |
   | RenderPass マージ | MergeRenderPasses() | スキップ |
   | クラッシュ時スタック | Execute() 内を指す | AddPass() 地点を指す |

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|--------|------|
| `FRDGUserValidation::ValidateAddPass()` | `RenderGraphValidation.h` | [[ref_rdg_validation]] パス追加の検証 |
| `FRDGUserValidation::ValidateExecutePassBegin()` | `RenderGraphValidation.h` | [[ref_rdg_validation]] ラムダ前検証 |
| `FRDGUserValidation::SetAllowRHIAccess()` | `RenderGraphValidation.h` | [[ref_rdg_validation]] GetRHI アクセス制御 |
| `FRDGBarrierValidation::ValidateBarrierBatchBegin()` | `RenderGraphValidation.h` | [[ref_rdg_validation]] バリアバッチ検証 |
| `FRDGTrace::AddResource()` | `RenderGraphTrace.h` | [[ref_rdg_trace]] リソース情報をトレース送信 |
| `FRDGTrace::AddTexturePassDependency()` | `RenderGraphTrace.h` | [[ref_rdg_trace]] テクスチャ依存をトレース送信 |
| `FRDGTrace::OutputGraphEnd()` | `RenderGraphTrace.h` | [[ref_rdg_trace]] グラフ統計をトレース送信 |
| `IF_RDG_ENABLE_DEBUG` | `RenderGraphDefinitions.h` | [[ref_rdg_definitions]] デバッグ条件コンパイルマクロ |
| `IF_RDG_ENABLE_TRACE` | `RenderGraphDefinitions.h` | [[ref_rdg_definitions]] トレース条件コンパイルマクロ |
