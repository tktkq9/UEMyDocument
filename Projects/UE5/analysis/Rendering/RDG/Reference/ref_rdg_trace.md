# リファレンス：RenderGraphTrace.h / RenderGraphTrace.cpp

- グループ: e - Debug
- 上位: [[e_rdg_debug_trace]]
- 関連: [[ref_rdg_validation]] | [[ref_rdg_builder]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphTrace.h`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphTrace.cpp`

## 概要

Unreal Insights へ RDG グラフ情報（リソース・パス依存関係）をトレース出力するクラス。  
`RDG_ENABLE_TRACE`（`UE_TRACE_ENABLED && !IS_PROGRAM && !UE_BUILD_SHIPPING`）でのみコンパイルされる。

---

## FRDGTrace

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `TransientAllocationStats` | `FRHITransientAllocationStats` | トランジェントアロケータのメモリ使用統計 |
| `GraphStartCycles` | `uint64` | グラフ開始時のサイクルカウンタ（duration 計算に使用） |
| `ResourceOrder` | `uint32` | リソース追加順序のカウンタ（Insights タイムライン用） |
| `bEnabled` | `bool` | `RDGChannel` トレースチャンネルが有効か |

### 公開メソッド

```cpp
// グラフ境界の出力
void OutputGraphBegin();                           // グラフ開始イベントを Insights へ送信
void OutputGraphEnd(const FRDGBuilder& GraphBuilder); // グラフ終了イベント + 統計を送信

// リソース / 依存関係の追跡
void AddResource(FRDGViewableResource* Resource);  // テクスチャ / バッファを記録
void AddTexturePassDependency(FRDGTexture* Texture, FRDGPass* Pass);
void AddBufferPassDependency(FRDGBuffer* Buffer,   FRDGPass* Pass);

// トレース有効確認
bool IsEnabled() const;  // RDGChannel && UE_TRACE_ENABLED
```

---

## トレースチャンネル

```cpp
// Unreal Insights への出力チャンネル
UE_TRACE_CHANNEL_EXTERN(RDGChannel, RENDERCORE_API);
```

Unreal Insights の「チャンネル設定」で `RDGChannel` を有効にすることでデータが記録される。  
コマンドライン引数 `-trace=RDG` または Insights から動的に有効化できる。

---

## 内部処理フロー

```
FRDGBuilder コンストラクタ
  └─ IF_RDG_ENABLE_TRACE(Trace.OutputGraphBegin())
      └─ GraphStartCycles を記録、IsEnabled() を評価

FRDGBuilder::CreateTexture() / CreateBuffer()
  └─ IF_RDG_ENABLE_TRACE(Trace.AddResource(Resource))
      └─ ResourceOrder をインクリメント
         → UE_TRACE: RDG_RESOURCE イベントを送信（名前・サイズ・フォーマット）

FRDGBuilder::SetupPassDependencies() ← AddPass から呼ばれる
  └─ IF_RDG_ENABLE_TRACE(Trace.AddTexturePassDependency(...) / AddBufferPassDependency(...))
      └─ UE_TRACE: RDG_DEPENDENCY イベント（リソース → パス の依存グラフ）

FRDGBuilder::Execute() 終了
  └─ IF_RDG_ENABLE_TRACE(Trace.OutputGraphEnd(GraphBuilder))
      └─ GPU 実行時間（GraphStartCycles から計算）
         TransientAllocationStats（トランジェントメモリ使用量）
         → UE_TRACE: RDG_GRAPH_END イベントを送信
```

---

## Insights での可視化内容

| データカテゴリ | 説明 |
|-------------|------|
| グラフ境界 | フレームごとの `FRDGBuilder` の開始〜終了タイムスタンプ |
| リソース一覧 | テクスチャ・バッファの名前・サイズ・フォーマット・追加順序 |
| パス依存グラフ | 各パスが読み書きするリソースの有向グラフ |
| トランジェント統計 | `FRHITransientAllocationStats`（ヒープ使用量・エイリアシング情報） |

---

## FRDGBuilder との連携

```cpp
class FRDGBuilder
{
#if RDG_ENABLE_TRACE
    FRDGTrace Trace;  // グラフのライフタイムと同期して常駐
#endif
};
```

`IF_RDG_ENABLE_TRACE(Op)` マクロ経由で各 API から自動呼び出しされる。  
ユーザーコードから直接呼ぶ必要はない。

---

> [!note]- RDGChannel を有効にする方法
> 1. **起動時**: コマンドライン引数 `-trace=default,RDG` を追加する
> 2. **Insights 接続後**: セッション設定の「Channels」タブで `RDGChannel` をチェック
> 3. **コンソールコマンド**: `Trace.Enable RDG` を実行
>
> なお `UE_BUILD_SHIPPING` ビルドでは `RDG_ENABLE_TRACE = 0` のため無効化される。

> [!note]- FRHITransientAllocationStats の内容
> `OutputGraphEnd` で収集される統計。トランジェントヒープの以下を含む:
> - ヒープ総数と各ヒープのサイズ
> - 各リソースの確保オフセット・サイズ・エイリアシング情報
> Insights の「RDG Transient Memory」ビューに表示される。  
> `r.RDG.Debug.ExtendResourceLifetimes` を有効にするとエイリアシングが無効化され、  
> 実際の最大メモリ使用量を確認できる。
