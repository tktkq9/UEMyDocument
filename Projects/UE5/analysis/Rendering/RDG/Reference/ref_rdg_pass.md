# リファレンス：RenderGraphPass.h / RenderGraphPass.cpp

- グループ: c - Pass
- 上位: [[c_rdg_pass]]
- 関連: [[ref_rdg_private]] | [[ref_rdg_builder]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphPass.h`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphPass.cpp`

## 概要

RDG パスの型定義・実行フラグ・並列化モードの全定義。  
`FRDGPass` 基底クラスと `TRDGLambdaPass` / `FRDGDispatchPass` の派生クラスを含む。  
バリアバッチ（`FRDGBarrierBatchBegin` / `FRDGBarrierBatchEnd`）の分割バリア機構も定義する。

---

## ERDGPassFlags

```cpp
enum class ERDGPassFlags : uint16 {
    None          = 0,
    Raster        = 1 << 0,   // グラフィクスパイプライン（描画）
    Compute       = 1 << 1,   // グラフィクスパイプライン上のコンピュート
    AsyncCompute  = 1 << 2,   // 非同期コンピュートパイプライン
    Copy          = 1 << 3,   // コピーコマンド（CopyTexture / CopyBuffer）
    NeverCull     = 1 << 4,   // カリング禁止（副作用あり）
    SkipRenderPass = 1 << 5,  // BeginRenderPass / EndRenderPass をスキップ
    NeverMerge    = 1 << 6,   // RenderPass マージ禁止
    NeverParallel = 1 << 7,   // レンダースレッド同期実行を強制

    // 複合フラグ
    Readback = Copy | NeverCull,  // GPU→CPU 読み戻し
};
```

### フラグ詳細

| フラグ | パイプライン | 備考 |
|-------|------------|------|
| `Raster` | Graphics | `BeginRenderPass` / `EndRenderPass` を自動発行 |
| `Compute` | Graphics | コンピュートシェーダ（同期） |
| `AsyncCompute` | AsyncCompute | 非同期コンピュートキュー。Graphics と並走可能 |
| `Copy` | Graphics | テクスチャ・バッファのコピー命令のみ |
| `NeverCull` | — | 参照されなくてもカリングされない。外部への副作用がある場合に使用 |
| `SkipRenderPass` | Graphics | `Raster` と組み合わせ。BeginRenderPass を自動発行しない |
| `NeverMerge` | Graphics | 前後パスの RenderPass とマージしない |
| `NeverParallel` | — | `ParallelExecute` モードでも同期（即時 CmdList）で実行 |
| `Readback` | Graphics | `Copy \| NeverCull` の短縮形 |

---

## ERDGPassTaskMode

```cpp
enum class ERDGPassTaskMode : uint8 {
    Inline,  // ラムダ引数: FRHICommandListImmediate& → レンダースレッドで同期実行
    Await,   // ラムダ引数: FRHICommandList& → Task で実行、Execute() 終了時に待機
    Async,   // ラムダ引数: FRDGAsyncTask, FRHICommandList& → Task で実行、手動待機
};
```

### ラムダ引数型 → TaskMode 対応

| ラムダ引数 | TaskMode | 並列可否 |
|-----------|---------|--------|
| `FRHICommandListImmediate&` | `Inline` | 不可（同期） |
| `FRHICommandList&` | `Await` | 可（Execute() 終了時に自動待機） |
| `FRHIComputeCommandList&` | `Await` | 可（Compute / AsyncCompute 専用） |
| `FRDGAsyncTask, FRHICommandList&` | `Async` | 可（ユーザーが `WaitForAsyncExecuteTask()` で待機） |

---

## ERDGBuilderFlags

```cpp
enum class ERDGBuilderFlags {
    None            = 0,
    ParallelSetup   = 1 << 0,  // AddSetupTask の並列化
    ParallelCompile = 1 << 1,  // グラフコンパイルの並列化
    ParallelExecute = 1 << 2,  // パス実行の並列化
    Parallel = ParallelSetup | ParallelCompile | ParallelExecute,
};
```

---

## FRDGPass（基底クラス）

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Name` | `FRDGEventName` | パス名（GPU プロファイラ / Insights に表示） |
| `ParameterStruct` | `FRDGParameterStruct` | パラメータ構造体への参照 |
| `Flags` | `ERDGPassFlags` | パスの種類・動作フラグ |
| `TaskMode` | `ERDGPassTaskMode` | 実行モード（Inline / Await / Async） |
| `Pipeline` | `ERHIPipeline` | Graphics / AsyncCompute |
| `Handle` | `FRDGPassHandle` | パスの一意ハンドル |
| `Workload` | `uint32` | 並列スケジューリング用の負荷ヒント（デフォルト 1） |
| `CrossPipelineProducer` | `FRDGPassHandle` | 最新のクロスパイプライン生成パス |
| `GraphicsForkPass` | `FRDGPassHandle` | AsyncCompute 区間の開始 Graphics パス |
| `GraphicsJoinPass` | `FRDGPassHandle` | AsyncCompute 区間の終了 Graphics パス |
| `PrologueBarrierPass` | `FRDGPassHandle` | Prologue バリアを担当するパス |
| `EpilogueBarrierPass` | `FRDGPassHandle` | Epilogue バリアを担当するパス |
| `NumTransitionsToReserve` | `uint32` | トランジション予約数の見積もり |
| `TextureStates` | `TArray<FTextureState>` | テクスチャとサブリソース状態のマップ |
| `BufferStates` | `TArray<FBufferState>` | バッファと状態のマップ |
| `Views` | `TArray<FRDGViewHandle>` | パスが参照する View ハンドル一覧 |
| `UniformBuffers` | `TArray<FRDGUniformBufferHandle>` | パスが参照する UniformBuffer ハンドル一覧 |
| `ExternalAccessOps` | `TArray<FExternalAccessOp>` | External Access 切り替えリスト |
| `ResourcesToBegin` | `TArray<FRDGPass*>` | このパス実行開始時に処理するリソース |
| `ResourcesToEnd` | `TArray<FRDGPass*>` | このパス実行終了時に処理するリソース |
| `Scope` | `FRDGScope*` | 所属する GPU イベントスコープ |
| `GPUMask` | `FRHIGPUMask` | マルチ GPU マスク（`WITH_MGPU` 時のみ） |

### ビットフィールド（PackedBits）

| フィールド | 説明 |
|----------|------|
| `bSkipRenderPassBegin` | RenderPass Begin をスキップ（マージ末尾パス） |
| `bSkipRenderPassEnd` | RenderPass End をスキップ（マージ先頭パス） |
| `bAsyncComputeBegin` | AsyncCompute 区間の先頭 |
| `bAsyncComputeEnd` | AsyncCompute 区間の末尾 |
| `bGraphicsFork` | AsyncCompute 区間の Fork Graphics パス |
| `bGraphicsJoin` | AsyncCompute 区間の Join Graphics パス |
| `bRenderPassOnlyWrites` | RenderPass 内でリソースを書き込みのみ（バリア最適化用） |
| `bSentinel` | 番兵パス（Prologue / Epilogue） |
| `bDispatchAfterExecute` | 実行後に RHI スレッドへディスパッチ |
| `bCulled` | カリング済みフラグ |
| `bEmptyParameters` | パラメータなし |
| `bHasExternalOutputs` | RDG が追跡しない外部 UAV あり |
| `bExternalAccessPass` | External Access トランジション専用パス |
| `bParallelExecuteBegin` | 並列実行スパンの開始 |
| `bParallelExecuteEnd` | 並列実行スパンの終了 |
| `bParallelExecute` | 並列実行中フラグ |

### 公開メソッド

```cpp
const TCHAR*         GetName()        const;
ERDGPassFlags        GetFlags()       const;
ERHIPipeline         GetPipeline()    const;  // Graphics / AsyncCompute
FRDGParameterStruct  GetParameters()  const;
FRDGPassHandle       GetHandle()      const;
uint32               GetWorkload()    const;
ERDGPassTaskMode     GetTaskMode()    const;

bool IsParallelExecuteAllowed() const;  // TaskMode != Inline
bool IsMergedRenderPassBegin()  const;  // !bSkipRenderPassBegin && bSkipRenderPassEnd
bool IsMergedRenderPassEnd()    const;  // bSkipRenderPassBegin && !bSkipRenderPassEnd
bool IsAsyncCompute()           const;
bool IsAsyncComputeBegin()      const;
bool IsAsyncComputeEnd()        const;
bool IsGraphicsFork()           const;
bool IsGraphicsJoin()           const;
bool IsCulled()                 const;
bool IsSentinel()               const;
FRDGPassHandle GetGraphicsForkPass() const;
FRDGPassHandle GetGraphicsJoinPass() const;
FRDGScope const* GetScope()          const;
FRHIGPUMask      GetGPUMask()        const;
```

---

## TRDGLambdaPass（ラムダ付きパス）

`AddPass()` が内部で生成する具体パス。ラムダ引数型から `ERDGPassTaskMode` を自動推定する。

```cpp
template <typename ParameterStructType, typename ExecuteLambdaType>
class TRDGLambdaPass : public FRDGPass
{
    // ラムダキャプチャサイズ上限: 1024 バイト（超えるとコンパイルエラー）
    static constexpr int32 kMaximumLambdaCaptureSize = 1024;
};
```

### TaskMode 推定ロジック

```cpp
// FRHICommandListImmediate& → Inline（他は全て Await または Async）
static constexpr ERDGPassTaskMode TaskMode =
    bIsCommandListImmediate ? ERDGPassTaskMode::Inline
    : bIsTaskAsync          ? ERDGPassTaskMode::Async
                            : ERDGPassTaskMode::Await;
```

---

## FRDGDispatchPass（並列コマンドリストパス）

`AddDispatchPass()` が内部で生成する。ラムダ内で複数の `FRHICommandListImmediate` を並列発行できる。

```cpp
class FRDGDispatchPass : public FRDGPass
{
    TArray<FRHICommandListImmediate::FQueuedCommandList> CommandLists;
    virtual void LaunchDispatchPassTasks(FRDGDispatchPassBuilder& DispatchPassBuilder) override;
};
```

---

## FRDGSentinelPass（番兵パス）

グラフの先頭（Prologue）と末尾（Epilogue）を表すダミーパス。  
実際のレンダリング命令は発行しないが、バリア伝播の起点・終点として機能する。

---

## 分割バリア（Split Barrier）機構

RDG は GPU パイプライン効率を上げるためバリアを「Begin」と「End」に分割して発行する。

### FRDGBarrierBatchBeginId

```cpp
struct FRDGBarrierBatchBeginId
{
    FRDGPassHandlesByPipeline Passes;       // Begin を発行するパスのパイプライン別ハンドル
    ERHIPipeline PipelinesAfter = ERHIPipeline::None; // End 側のパイプライン
};
```

### FRDGTransitionInfo（ビットパック構造体）

```cpp
struct FRDGTransitionInfo
{
    uint64 AccessBefore            : 21;  // 遷移前の ERHIAccess
    uint64 AccessAfter             : 21;  // 遷移後の ERHIAccess
    uint64 ResourceHandle          : 16;  // テクスチャ / バッファのハンドル
    uint64 ResourceType            : 3;   // ERDGViewableResourceType
    uint64 ResourceTransitionFlags : 3;   // EResourceTransitionFlags

    union {
        struct { uint16 ArraySlice; uint8 MipIndex; uint8 PlaneSlice; } Texture;
        struct { uint64 CommitSize; } Buffer;
    };
};
```

### FRDGBarrierBatchBegin

複数リソースの遷移を一括管理し、RHI へ一度に Submit する。

```cpp
class FRDGBarrierBatchBegin
{
public:
    void AddTransition(FRDGViewableResource* Resource, FRDGTransitionInfo Info);
    void AddAlias(FRDGViewableResource* Resource, const FRHITransientAliasingInfo& Info);
    void CreateTransition(TConstArrayView<FRHITransitionInfo> TransitionsRHI);
    void Submit(FRHIComputeCommandList& RHICmdList, ERHIPipeline Pipeline);
    void SetUseCrossPipelineFence(bool bUseSeparateTransition);

private:
    const FRHITransition* Transition = nullptr;
    const FRHITransition* SeparateFenceTransition = nullptr;
    TArray<FRDGTransitionInfo> Transitions;
    TArray<FRHITransientAliasingInfo> Aliases;
    ERHITransitionCreateFlags TransitionFlags;
    ERHIPipeline PipelinesToBegin;
    ERHIPipeline PipelinesToEnd;
};
```

### FRDGBarrierBatchEnd

Begin バッチへの依存を登録し、End 側で一括 Submit する。

```cpp
class FRDGBarrierBatchEnd
{
public:
    void AddDependency(FRDGBarrierBatchBegin* BeginBatch);
    void Submit(FRHIComputeCommandList& RHICmdList, ERHIPipeline Pipeline);

private:
    TArray<FRDGBarrierBatchBegin*> Dependencies;
    FRDGPass*          Pass;
    ERDGBarrierLocation BarrierLocation;
};
```

### パスごとのバリアバッチ配置

| バッチポインタ（FRDGPass メンバ） | 発行タイミング | 目的 |
|-------------------------------|-------------|------|
| `PrologueBarriersToBegin` | パス実行前 Begin | 前パスからの遷移を開始 |
| `PrologueBarriersToEnd` | パス実行前 End | Begin を完了させる |
| `EpilogueBarriersToBeginForGraphics` | パス実行後 Begin | Graphics パイプライン向け遷移を開始 |
| `EpilogueBarriersToBeginForAsyncCompute` | パス実行後 Begin | AsyncCompute 向け遷移を開始 |
| `EpilogueBarriersToBeginForAll` | パス実行後 Begin | 両パイプライン共通の遷移を開始 |
| `EpilogueBarriersToEnd` | パス実行後 End | Epilogue Begin を完了させる |

---

## FRDGPass 内部の状態構造体

### FTextureState

```cpp
struct FTextureState {
    FRDGTextureRef Texture = nullptr;
    FRDGTextureSubresourceState State;       // このパスでの期待アクセス状態
    FRDGTextureSubresourceState MergeState;  // マージされたパスの状態
    uint32 ReferenceCount = 0;
};
```

### FBufferState

```cpp
struct FBufferState {
    FRDGBufferRef Buffer = nullptr;
    FRDGSubresourceState State;
    FRDGSubresourceState* MergeState = nullptr;
    uint32 ReferenceCount = 0;
};
```

---

## その他の列挙型

### ERDGSetupTaskWaitPoint

```cpp
enum class ERDGSetupTaskWaitPoint : uint8 {
    Compile,   // コンパイル前に待機（デフォルト）
    Execute,   // 実行直前まで待機
};
```

### ERDGViewableResourceType

```cpp
enum class ERDGViewableResourceType : uint8 {
    Texture,
    Buffer,
    MAX,
};
```

### ERDGViewType

```cpp
enum class ERDGViewType : uint8 {
    TextureUAV,
    TextureSRV,
    BufferUAV,
    BufferSRV,
    MAX,
};
```

### ERDGUnorderedAccessViewFlags

```cpp
enum class ERDGUnorderedAccessViewFlags : uint8 {
    None        = 0,
    SkipBarrier = 1 << 0,  // 同一パス内の複数 UAV 書き込みでバリアをスキップ
};
```

---

> [!note]- ERDGBarrierLocation（バリア挿入位置）
> ```cpp
> enum class ERDGBarrierLocation : uint8 {
>     Prologue,  // パス実行前に End バリアを発行
>     Epilogue,  // パス実行後に End バリアを発行（デフォルト）
> };
> ```
> Async Compute の Fork/Join 周辺でクロスパイプラインフェンスが必要な場合に Prologue が選択される。

> [!note]- FRDGPassHandlesByPipeline
> ```cpp
> using FRDGPassHandlesByPipeline  = TRHIPipelineArray<FRDGPassHandle>;
> using FRDGPassesByPipeline       = TRHIPipelineArray<FRDGPass*>;
> ```
> `TRHIPipelineArray<T>` は `ERHIPipeline::Graphics` / `AsyncCompute` をインデックスとする固定長配列。  
> `FRDGBarrierBatchBeginId::Passes` に使われ、バリア Begin のパイプライン別スケジューリングに用いる。

> [!note]- TRDGLambdaPass キャプチャ上限（1024 バイト）
> ラムダのキャプチャサイズが 1024 バイトを超えるとコンパイルエラーになる。  
> 大量のデータを渡す場合は `TSharedPtr` 等をキャプチャし、実態を別途確保する。

> [!note]- AddDispatchPass とスレッドセーフ
> `FRDGDispatchPassBuilder` は `FRDGBuilder::AddDispatchPass()` から受け取り、  
> `AddDispatch()` を複数スレッドから安全に呼び出せる。  
> 発行された `FRHICommandListImmediate` はパス完了後にマージされる。
