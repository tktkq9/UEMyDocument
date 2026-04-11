# リファレンス：RenderGraphEvent.h / RenderGraphEvent.inl / RenderGraphEvent.cpp

- グループ: d - Parameters
- 上位: [[d_rdg_parameters]]
- 関連: [[ref_rdg_builder]] | [[ref_rdg_utils]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphEvent.h`
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphEvent.inl`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphEvent.cpp`

## 概要

GPU プロファイラ（RenderDoc / PIX / Unreal Insights）に表示される  
イベント名とスコープを管理するクラス群・マクロ群。  
ビルド構成によって文字列処理コストがゼロになるよう設計されている。

---

## FRDGEventName

GPU イベント名を格納するクラス。`AddPass` の第一引数として使用する。

```cpp
class FRDGEventName final
{
public:
    FRDGEventName() = default;

    // 書式付きイベント名（printf スタイル）
    explicit FRDGEventName(const TCHAR* EventFormat, ...);

    // 非書式版（書式処理を省略）
    FRDGEventName(int32 NonVariadic, const TCHAR* EventName);

    // 内部文字列を取得（デバッグ用）
    const TCHAR* GetTCHAR() const;

    // 書式展開済みか（RDG_EVENTS_STRING_COPY ビルドのみ true）
    bool HasFormattedString() const;
};
```

### ビルド構成による挙動の違い

| `RDG_EVENTS` 値 | 文字列処理 | 用途 |
|----------------|-----------|------|
| `RDG_EVENTS_NONE` | なし | Shipping ビルドでゼロコスト |
| `RDG_EVENTS_STRING_REF` | `const TCHAR*` 保持のみ | Test ビルド、RHI Breadcrumbs |
| `RDG_EVENTS_STRING_COPY` | `FString` に展開・保持 | Development / Debug |

---

## イベント名マクロ

```cpp
// AddPass の第一引数として使用（書式文字列対応）
RDG_EVENT_NAME("MyPass")
RDG_EVENT_NAME("Bloom %dx%d", Width, Height)
```

---

## スコープマクロ

### GPU イベントスコープ

```cpp
// プロファイラに階層スコープを追加
RDG_EVENT_SCOPE(GraphBuilder, "PostProcess");
{
    RDG_EVENT_SCOPE(GraphBuilder, "Bloom");
    GraphBuilder.AddPass(RDG_EVENT_NAME("Setup"), ...);
    GraphBuilder.AddPass(RDG_EVENT_NAME("Downsample"), ...);
}
```

### ERDGScopeFlags

```cpp
enum class ERDGScopeFlags : uint8
{
    None  = 0,
    Final = 1 << 0,   // 同型のネストスコープを無効化
    // Always = 1 << 1,  // CVar によるスコープ無効化を無視して常に出力
};
```

---

## GPU Stat スコープ

```cpp
// r.GPU.StatGroup と連動したプロファイルスコープ
RDG_GPU_STAT_SCOPE(GraphBuilder, Bloom);

// RDG_GPU_STAT_SCOPE_VERBOSE は一部ビルドでのみ有効
```

---

## CPU プロファイルスコープ

```cpp
// CSV プロファイラに記録
RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, PostProcess);

// Insights の CPU トレースに記録
RDG_CPU_STAT_SCOPE(GraphBuilder, Bloom);
```

---

## FRDGScopeState（内部型）

スコープのスタック状態を管理する内部クラス。  
`FRDGBuilder` が所有し、`AddPass` 時にアクティブなスコープを各パスに記録する。

---

## 使用例：入れ子スコープの全パターン

```cpp
{
    RDG_EVENT_SCOPE(GraphBuilder, "RenderSystem");
    {
        RDG_GPU_STAT_SCOPE(GraphBuilder, LumenGI);
        RDG_EVENT_SCOPE(GraphBuilder, "Lumen");
        GraphBuilder.AddPass(RDG_EVENT_NAME("RadianceCache Update"), ...);
        GraphBuilder.AddPass(RDG_EVENT_NAME("Screen Probe Gather"), ...);
    }
    {
        RDG_GPU_STAT_SCOPE(GraphBuilder, Reflections);
        RDG_EVENT_SCOPE(GraphBuilder, "Reflections");
        GraphBuilder.AddPass(RDG_EVENT_NAME("TraceReflections"), ...);
        GraphBuilder.AddPass(RDG_EVENT_NAME("Resolve"), ...);
    }
}
```
