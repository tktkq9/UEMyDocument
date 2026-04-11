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

```cpp
#if RDG_ENABLE_TRACE

class FRDGTrace
{
public:
    FRDGTrace();

    // ──── グラフ境界 ────
    // FRDGBuilder コンストラクタ / デストラクタから呼ばれる
    void OutputGraphBegin();
    void OutputGraphEnd(const FRDGBuilder& GraphBuilder);

    // ──── リソース登録 ────
    // FRDGBuilder::CreateTexture / CreateBuffer から呼ばれる
    void AddResource(FRDGViewableResource* Resource);

    // ──── 依存関係追跡 ────
    // AddPass 時にパラメータ解析で呼ばれる
    void AddTexturePassDependency(FRDGTexture* Texture, FRDGPass* Pass);
    void AddBufferPassDependency(FRDGBuffer* Buffer, FRDGPass* Pass);

    // トレースが有効か確認
    bool IsEnabled() const;

    // Transient アロケーション統計
    FRHITransientAllocationStats TransientAllocationStats;

private:
    uint64 GraphStartCycles{};   // グラフ開始サイクルカウンタ
    uint32 ResourceOrder{};      // リソース追加順序カウンタ
    bool bEnabled;               // チャンネルが有効か
};

#endif // RDG_ENABLE_TRACE
```

---

## トレースチャンネル

```cpp
// Unreal Insights に RDG データを送るトレースチャンネル
UE_TRACE_CHANNEL_EXTERN(RDGChannel, RENDERCORE_API);
```

Unreal Insights の接続設定で `RDGChannel` を有効にすることでデータが記録される。

---

## Insights での可視化内容

| データ | 説明 |
|-------|------|
| グラフ境界 | フレームごとの `FRDGBuilder` の開始〜終了 |
| リソース一覧 | テクスチャ・バッファの名前・サイズ・フォーマット |
| パス依存 | 各パスが読み書きするリソースのグラフ |
| Transient 統計 | トランジェントアロケータのメモリ使用量 |

---

## 使用箇所

```cpp
class FRDGBuilder
{
    // ...
#if RDG_ENABLE_TRACE
    FRDGTrace Trace;
#endif
};
```

`FRDGBuilder` のコンストラクタ・デストラクタ・`AddPass`・`CreateTexture` から自動的に呼ばれる。  
ユーザーコードから直接呼ぶ必要はない。
