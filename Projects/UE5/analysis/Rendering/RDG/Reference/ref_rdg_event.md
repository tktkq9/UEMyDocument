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

### メンバ変数

| 変数 | 型 | 存在条件 | 説明 |
|-----|-----|---------|------|
| `EventFormat` | `const TCHAR*` | `RDG_EVENTS >= STRING_REF` | フォーマット文字列（デバッグ用） |
| `FormattedEventName` | `FString` | `RDG_EVENTS == STRING_COPY` | printf 展開済み文字列 |

### コンストラクタ

```cpp
// 書式付きイベント名（printf スタイル）
explicit FRDGEventName(const TCHAR* EventFormat, ...);

// 静的文字列（書式処理を省略、軽量）
FRDGEventName(int32 NonVariadic, const TCHAR* EventName);
```

### 公開メソッド

```cpp
const TCHAR* GetTCHAR() const;           // 文字列ポインタを取得（デバッグ用）
bool HasFormattedString() const;          // STRING_COPY ビルドのみ true
```

### ビルド構成による挙動

| `RDG_EVENTS` 値 | メンバ変数 | 文字列処理コスト |
|----------------|----------|--------------|
| `RDG_EVENTS_NONE` (0) | なし | ゼロ（Shipping） |
| `RDG_EVENTS_STRING_REF` (1) | `EventFormat` のみ | 最小（ポインタ保持のみ） |
| `RDG_EVENTS_STRING_COPY` (2) | `EventFormat` + `FormattedEventName` | printf フォーマット実行 |

---

## ERDGScopeFlags

```cpp
enum class ERDGScopeFlags : uint8
{
    None        = 0,
    Final       = 1 << 0,   // 同型のネストスコープを無効化（最外層のみ出力）
    AlwaysEnable = 1 << 1,  // CVar によるスコープ無効化を無視して常に出力
    Stat        = 1 << 2,   // GPU Stat を含む（スコープ無効化でも Stat 部分は有効）
};
ENUM_CLASS_FLAGS(ERDGScopeFlags);
```

---

## イベント名マクロ

```cpp
// AddPass の第一引数として使用（書式文字列対応）
RDG_EVENT_NAME("MyPass")                 // 静的文字列（高速）
RDG_EVENT_NAME("Bloom %dx%d", W, H)     // 動的文字列（フォーマット展開）
```

---

## スコープマクロ

### GPU イベントスコープ（FRDGScope_RHI）

```cpp
// プロファイラに階層スコープを追加（RenderDoc / PIX / Insights のタイムラインに表示）
RDG_EVENT_SCOPE(GraphBuilder, "PostProcess");
{
    RDG_EVENT_SCOPE(GraphBuilder, "Bloom");
    GraphBuilder.AddPass(RDG_EVENT_NAME("Setup"), ...);
    GraphBuilder.AddPass(RDG_EVENT_NAME("Downsample"), ...);
}
```

### GPU Stat スコープ（FRDGScope_GPU）

```cpp
// r.GPU.StatGroup と連動したプロファイルスコープ
// HAS_GPU_STATS が有効なビルドでのみ実効
RDG_GPU_STAT_SCOPE(GraphBuilder, Bloom);
RDG_GPU_STAT_SCOPE_VERBOSE(GraphBuilder, Bloom);  // 一部ビルドのみ有効
```

### CPU プロファイルスコープ

```cpp
// CSV プロファイラに記録（FRDGScope_CSVExclusive）
RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, PostProcess);

// CPU プロファイラトレース（FRDGScope_CPU）
RDG_CPU_STAT_SCOPE(GraphBuilder, Bloom);
```

### Dynamic Render Scaling バジェットスコープ（FRDGScope_Budget）

```cpp
// 動的解像度スケーリングのバジェット計測スコープ
RDG_BUDGET_SCOPE(GraphBuilder, MyBudget);
```

---

## スコープ実装クラス

各スコープ型は以下のコールバックメソッドを実装する：

| メソッド | 呼び出しタイミング | スレッド |
|---------|----------------|---------|
| `Constructor` / `ImmediateEnd()` | グラフ構築時（即時モード） | レンダースレッド |
| `BeginCPU()` / `EndCPU()` | パスラムダ実行時 | 並列タスクスレッド |
| `BeginGPU()` / `EndGPU()` | GPU パイプライン別に 1 回 | 並列タスクスレッド |

### FRDGScope_RHI

RHI ブレッドクラムノードを管理する。CPU と GPU のタイムラインを同期させる。

```cpp
class FRDGScope_RHI {
    FRHIBreadcrumbNode* Node;  // プロファイラ表示用ノード

    // BeginCPU: BeginBreadcrumbCPU → BeginBreadcrumbGPU（非プリスコープ時）
    // EndCPU:   EndBreadcrumbGPU → EndBreadcrumbCPU（非プリスコープ時）
    // BeginGPU/EndGPU: RHI API が自動修正するため何もしない
};
```

### FRDGScope_GPU（レガシー GPU プロファイラ）

```cpp
struct FRDGScope_GPU {
    FRealtimeGPUProfilerQuery StartQuery;
    FRealtimeGPUProfilerQuery StopQuery;
    FName StatName;
    TStatId StatId;
    FString StatDescription;
    FRHIDrawStatsCategory const* CurrentCategory;
};
```

---

## FRDGScope（スコープツリーノード）

全スコープ型の共通ラッパー。スコープのツリー構造を形成する。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Parent` | `FRDGScope* const` | 親スコープ（null = ルート） |
| `CPUFirstPass` | `FRDGPass*` | スコープ内最初の CPU パス |
| `CPULastPass` | `FRDGPass*` | スコープ内最後の CPU パス |
| `GPUFirstPass` | `TRHIPipelineArray<FRDGPass*>` | パイプラインごとの最初のパス |
| `GPULastPass` | `TRHIPipelineArray<FRDGPass*>` | パイプラインごとの最後のパス |

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

---

> [!note]- ERDGScopeFlags::Final の使いどころ
> `Final` フラグを付けると同型の内側スコープが無効化される。  
> たとえば外側に `RDG_EVENT_SCOPE_FINAL(GraphBuilder, "Lumen")` を使うと  
> 内側の `RDG_EVENT_SCOPE` が出力されなくなり、プロファイラのネストが平坦化される。  
> 外部から渡された GraphBuilder に追加するサブシステムで使うと効果的。

> [!note]- RDG_EVENTS_STRING_COPY と RDG_EVENTS_STRING_REF の区別
> - `STRING_REF`: `const TCHAR*` を保持するだけ。`RDG_EVENT_NAME("Pass %d", n)` では  
>   フォーマット展開が起きない。静的文字列のみ有効（ポインタが無効化されない保証が必要）。
> - `STRING_COPY`: `FString` に printf 展開する。動的な数値入りイベント名が正しく表示される。  
>   Development / Debug ビルドでのみ有効。

> [!note]- FRDGScope_Budget の用途
> `DynamicRenderScaling::FBudget` スコープと連動し、その配下のパスの GPU 時間を予算として計測する。  
> `r.DynamicRes` 系の動的解像度スケーリング調整に使われる。
