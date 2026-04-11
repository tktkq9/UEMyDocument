# リファレンス：RenderGraphPrivate.h / RenderGraphPrivate.cpp

- グループ: c - Pass
- 上位: [[c_rdg_pass]]
- 関連: [[ref_rdg_pass]] | [[ref_rdg_builder]]
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphPrivate.h`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphPrivate.cpp`

## 概要

RDG のデバッグ・検証用 CVar と内部グローバル変数を定義するプライベートヘッダ / 実装ファイル。  
外部公開されていない内部専用ヘッダで、`FRDGBuilder` や `FRDGUserValidation` から `#include` される。  
すべてのデバッグ CVar は `RDG_ENABLE_DEBUG`（非 Shipping / 非 Test）でのみ実変数になる。

---

## CVar 一覧

### デバッグ実行制御（RDG_ENABLE_DEBUG 限定）

| CVar | グローバル変数 | デフォルト | 説明 |
|------|-------------|----------|------|
| `r.RDG.ImmediateMode` | `GRDGImmediateMode` | 0 | パスを宣言と同時に即時実行する。クラッシュ時のコールスタックをたどるのに有用 |
| `r.RDG.Validation` | `GRDGValidation` | 1 | API 呼び出しとパス依存関係の正当性を検証する |
| `r.RDG.Debug.FlushGPU` | `GRDGDebugFlushGPU` | 0 | 各パス実行後に GPU をフラッシュ。AsyncCompute と ParallelExecute を自動無効化 |
| `r.RDG.Debug.ExtendResourceLifetimes` | `GRDGDebugExtendResourceLifetimes` | 0 | リソースライフタイムを延長してエイリアシングを無効化（エイリアシング起因のバグ切り分けに使用） |
| `r.RDG.Debug.DisableTransientResources` | `GRDGDebugDisableTransientResources` | 0 | トランジェントアロケータからリソースを除外。フィルタは `r.RDG.Debug.ResourceFilter` |
| `r.RDG.ClobberResources` | `GRDGClobberResources` | 0 | 未初期化リソースに特定値を書き込む（下記参照） |
| `r.RDG.OverlapUAVs` | `GRDGOverlapUAVs` | 1 | UAV バリアを必要最小限に抑える（0 = 常にバリア挿入） |
| `r.RDG.TransitionLog` | `GRDGTransitionLog` | 0 | リソーストランジションをコンソールにログ出力（>0: N フレーム、<0: 無期限） |

### ClobberResources の値

| 値 | RGBA 値 | 用途 |
|----|--------|------|
| 0 | 無効 | 通常動作 |
| 1 | 1000.0f | 未初期化検出（大きすぎる値） |
| 2 | NaN | NaN 伝播検出 |
| 3 | +∞ | Inf 伝播検出 |

Depth は固定値 `0.123456789f`、Stencil は `123` が書き込まれる。

### グラフ・パス・リソースフィルタ

| CVar | 説明 |
|------|------|
| `r.RDG.Debug.GraphFilter` | デバッグイベントを特定グラフ名に限定（`"None"` でリセット） |
| `r.RDG.Debug.PassFilter` | デバッグイベントを特定パス名に限定（`"None"` でリセット） |
| `r.RDG.Debug.ResourceFilter` | デバッグイベントを特定リソース名に限定（`"None"` でリセット） |

フィルタ文字列が `!` で始まると **反転マッチ**（その名前を含まないものを対象にする）。

### 通常ビルドでも有効な制御 CVar

| CVar | グローバル変数 | デフォルト | 説明 |
|------|-------------|----------|------|
| `r.RDG.AsyncCompute` | `GRDGAsyncCompute` | 1 | AsyncCompute ポリシー（0: 無効、1: タグ付きのみ、2: 全 Compute） |
| `r.RDG.CullPasses` | `GRDGCullPasses` | 1 | 未使用出力を持つパスのカリング |
| `r.RDG.MergeRenderPasses` | `GRDGMergeRenderPasses` | 1 | 隣接する同一レンダーターゲットの RenderPass をマージ |
| `r.RDG.TransientAllocator` | `GRDGTransientAllocator` | 1 | トランジェントアロケータの使用（0: 無効、1: 有効、2: FastVRAM のみ） |
| `r.RDG.TransientExtractedResources` | `GRDGTransientExtractedResources` | 1 | 抽出リソースのトランジェント化（0: 無効、2: 強制有効） |
| `r.RDG.AsyncComputeTransientAliasing` | `GRDGAsyncComputeTransientAliasing` | 1 | AsyncCompute リソースを Graphics ヒープとエイリアス（RHI 対応が必要） |

### 並列実行制御 CVar（RDG_ENABLE_PARALLEL_TASKS 限定）

| CVar | グローバル変数 | デフォルト | 説明 |
|------|-------------|----------|------|
| `r.RDG.ParallelDestruction` | `GRDGParallelDestruction` | 1 | グラフの解体を非同期タスクで実行 |
| `r.RDG.ParallelSetup` | `GRDGParallelSetup` | 1 | `AddSetupTask` のタスクを並列起動 |
| `r.RDG.ParallelCompile` | `GRDGParallelCompile` | 1 | グラフコンパイルをタスクで並列実行 |
| `r.RDG.AsyncSetupQueue` | `GRDGAsyncSetupQueue` | 1 | `FlushSetupQueue` 時にパスセットアップを非同期処理 |
| `r.RDG.ParallelSetup.TaskPriorityBias` | `GRDGParallelSetupTaskPriorityBias` | 1 | セットアップタスクの優先度バイアス |
| `r.RDG.ParallelExecute` | `GRDGParallelExecute` | 2 | 並列実行モード（0: 無効、1: 全タスク待機、2: 非同期タスクあり） |
| `r.RDG.ParallelExecute.PassMin` | `GRDGParallelExecutePassMin` | 1 | 並列化する最小パス連続数 |
| `r.RDG.ParallelExecute.PassMax` | `GRDGParallelExecutePassMax` | 32 | 並列化する最大パス連続数 |
| `r.RDG.ParallelExecute.PassTaskModeThreshold` | `GRDGParallelExecutePassTaskModeThreshold` | 2 | TaskMode 不一致時のバッチフラッシュ閾値 |
| `r.RDG.ParallelExecuteStress` | `GRDGParallelExecuteStress` | 0 | ストレステスト（1 パスあたり 1 タスク、RenderPass マージ無効） |

---

## 内部ヘルパー関数

```cpp
// フィルタ文字列に Name がマッチするか確認（! 前置で反転）
bool IsDebugAllowedForGraph(const TCHAR* GraphName);
bool IsDebugAllowedForPass(const TCHAR* PassName);
bool IsDebugAllowedForResource(const TCHAR* ResourceName);

// ClobberResources の値に対応した RGBA / Buffer / Depth / Stencil 値を返す
FLinearColor GetClobberColor();
uint32       GetClobberBufferValue();
float        GetClobberDepth();      // 0.123456789f
uint8        GetClobberStencil();    // 123

// RDG 警告をコンソールに出力
void EmitRDGWarning(const FString& WarningMessage);
```

---

## FRDGAllowRHIAccessScope

`bAllowRHIAccess` フラグをスコープ内で一時的に true にする RAII クラス。  
`FRDGBuilder` が Execute ループ内でパスラムダを呼び出す際に使用する。

```cpp
// 通常、パスラムダ外で RHI リソースに直接アクセスすると検証エラーになる。
// このスコープ内でのみアクセスが許可される。
class FRDGAllowRHIAccessScope
{
public:
    FRDGAllowRHIAccessScope()  { GRDGAllowRHIAccess = true; }
    ~FRDGAllowRHIAccessScope() { GRDGAllowRHIAccess = false; }
};

// 使用例（FRDGBuilder::ExecutePass 内）
#define RDG_ALLOW_RHI_ACCESS_SCOPE() \
    FRDGAllowRHIAccessScope RDGAllowRHIAccessScopeRAII;
```

---

## プリプロセッサマクロ

```cpp
// DumpGraph の詳細レベル
#define RDG_DUMP_GRAPH_PRODUCERS 1  // プロデューサー依存のみ
#define RDG_DUMP_GRAPH_RESOURCES 2  // リソースも含む
#define RDG_DUMP_GRAPH_TRACKS    3  // タイムライン形式

// AsyncCompute ポリシー値（GRDGAsyncCompute に使用）
#define RDG_ASYNC_COMPUTE_DISABLED      0
#define RDG_ASYNC_COMPUTE_ENABLED       1
#define RDG_ASYNC_COMPUTE_FORCE_ENABLED 2
```

---

## 統計カウンタ（STAT_RDG_* / COUNTER_RDG_*）

`STATGROUP_RDG` グループで Unreal Stats と Unreal Insights の両方に出力される。

| 統計名 | 種別 | 説明 |
|-------|------|------|
| `Passes` | Dword | 総パス数 |
| `Passes Culled` | Dword | カリングされたパス数 |
| `Render Passes Merged` | Dword | マージされた RenderPass 数 |
| `Pass Dependencies` | Dword | パス間依存数 |
| `Textures` / `Buffers` | Dword | 登録テクスチャ / バッファ数 |
| `Transient Textures` / `Buffers` | Dword | トランジェントリソース数 |
| `Resource Transitions` | Dword | バリアトランジション数 |
| `Resource Acquires and Discards` | Dword | エイリアシング操作数 |
| `Builder Watermark` | Memory | ピークメモリ使用量 |
| `Setup` / `Compile` / `Execute` | Cycle | 各フェーズの CPU 時間 |

---

> [!note]- GRDGAllowRHIAccess と GRDGAllowRHIAccessAsync の使い分け
> - `GRDGAllowRHIAccess`：レンダースレッドからパスラムダ実行中に RHI リソースへアクセスする際に true になる
> - `GRDGAllowRHIAccessAsync`：並列実行タスク内で同様のアクセスを許可する際に true になる
> - `FRDGUserValidation` はこれらを参照し、パスラムダ外での不正 RHI アクセスを検出する

> [!note]- r.RDG.Debug.FlushGPU の副作用
> このフラグを有効にすると `GRDGAsyncCompute = 0` と `GRDGParallelExecute = 0` が自動設定される。  
> これはバリアの分割（Split Barrier）が GPU フラッシュ後には意味を持たなくなるためで、  
> `CVarRDGAsyncComputeSink` のコールバック内で強制上書きされる。

> [!note]- r.RDG.ParallelExecuteStress の動作
> 有効化すると `GRDGMergeRenderPasses = 0` / `PassMin = 1` / `PassMax = 1` に強制設定し、  
> 1 パスあたり 1 タスクを起動する。無効化時は元の値に復元される。  
> CI 環境での並列バグ検出に使用する。
