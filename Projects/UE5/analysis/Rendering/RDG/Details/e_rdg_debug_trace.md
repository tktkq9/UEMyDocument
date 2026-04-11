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
