# リファレンス：RenderGraphPrivate.cpp

- グループ: c - Pass
- 上位: [[c_rdg_pass]]
- 関連: [[ref_rdg_pass]] | [[ref_rdg_builder]]
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphPrivate.cpp`

## 概要

RDG のデバッグ・検証用 CVar を定義するファイル。  
グラフコンパイル・実行ループの内部ヘルパー関数も含む（`RenderGraphPrivate.h`）。  
すべての CVar は `RDG_ENABLE_DEBUG`（非 Shipping / 非 Test）でのみアクティブ。

---

## 主要 CVar 一覧

### デバッグ実行制御

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RDG.ImmediateMode` | 0 | パスを宣言と同時に即時実行する。クラッシュ時のコールスタックを追うのに有用 |
| `r.RDG.Validation` | 1 | API 呼び出しとパス依存関係の正当性を検証する |
| `r.RDG.Debug.FlushGPU` | 0 | 各パス実行後に GPU をフラッシュ（AsyncCompute と並列実行を自動無効化） |

### トランジェントリソース制御

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RDG.Debug.ExtendResourceLifetimes` | 0 | リソースライフタイムを延長してエイリアシングを無効化（エイリアシング起因のバグ切り分けに使用） |
| `r.RDG.Debug.DisableTransientResources` | 0 | トランジェントアロケータからリソースを除外する（フィルタは `r.RDG.Debug.ResourceFilter`） |
| `r.RDG.ClobberResources` | 0 | 未初期化リソースをランダム値で上書き（初期化漏れの検出に有用） |

### ダンプ・可視化

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RDG.Debug.ResourceFilter` | `""` | 特定リソース名をフィルタ（`ExtendResourceLifetimes` 等に適用） |
| `r.RDG.DumpGraph` | 0 | グラフ構造を Dot ファイルにダンプ（0 = 無効, 1 = 次フレームのみ） |

---

## RenderGraphPrivate.h — 内部型

`RenderGraphPrivate.h` は外部公開されていない内部ヘッダ。  
以下の内部型・関数を定義する:

| 型 / 関数 | 説明 |
|-----------|------|
| `FRDGBarrierBatchBegin` | バリア一括発行の開始バッチ |
| `FRDGBarrierBatchEnd` | バリア一括発行の終了バッチ |
| `FRDGPassRegistry` | グラフ内パスの登録管理 |
| `FRDGTextureSubresourceState` | テクスチャサブリソースのアクセス状態（`ERHIAccess` 配列） |
| `FRDGBufferSubresourceState` | バッファのアクセス状態 |

---

## グローバル変数

```cpp
// デバッグ用: DumpGraph 等で使われる匿名グラフのカウンタ
int32 GRDGDumpGraphUnknownCount = 0;
```
