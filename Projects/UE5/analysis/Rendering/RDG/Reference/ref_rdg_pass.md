# REF: RDG パス型・フラグ

- 対象ファイル: `RenderGraphPass.h`, `RenderGraphDefinitions.h`
- 関連Details: [[c_rdg_pass]]

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

---

## ERDGPassTaskMode

```cpp
enum class ERDGPassTaskMode : uint8 {
    // ラムダ引数: FRHICommandListImmediate&
    Inline,  // レンダースレッドで同期実行

    // ラムダ引数: FRHICommandList&
    Await,   // Task グラフで実行、Execute() 終了時に待機

    // ラムダ引数: FRDGAsyncTask, FRHICommandList&
    Async,   // Task グラフで実行、ユーザーが手動で待機
};
```

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

```cpp
class FRDGPass
{
public:
    const TCHAR*            GetName()        const;
    ERDGPassFlags           GetFlags()       const;
    ERHIPipeline            GetPipeline()    const;  // Graphics / AsyncCompute
    FRDGParameterStruct     GetParameters()  const;
    FRDGPassHandle          GetHandle()      const;
    uint32                  GetWorkload()    const;
    ERDGPassTaskMode        GetTaskMode()    const;

    bool IsParallelExecuteAllowed() const;
    bool IsMergedRenderPassBegin()  const;
    bool IsMergedRenderPassEnd()    const;
    bool IsAsyncCompute()           const;
    bool IsCulled()                 const;
    bool IsSetupOnRenderThread()    const;

protected:
    FRDGPass(FRDGEventName&&, FRDGParameterStruct, ERDGPassFlags, ERDGPassTaskMode);
};
```

---

## TRDGLambdaPass（ラムダ付きパス）

```cpp
// AddPass() が内部で生成する具体パス
template <typename ParameterStructType, typename ExecuteLambdaType>
class TRDGLambdaPass : public FRDGPass
{
    // ExecuteLambdaType の引数型から ERDGPassTaskMode を推定:
    //   FRHICommandListImmediate& → Inline
    //   FRHICommandList&          → Await
    //   FRDGAsyncTask, FRHICommandList& → Async
};
```

---

## FRDGDispatchPass（並列コマンドリストパス）

```cpp
// AddDispatchPass() が内部で生成する
class FRDGDispatchPass : public FRDGPass
{
    TArray<FRHICommandListImmediate::FQueuedCommandList> CommandLists;
};
```

---

## FRDGSentinelPass（番兵パス）

```cpp
// グラフの先頭と末尾を表すダミーパス
class FRDGSentinelPass : public FRDGPass { ... };
```

---

## ERDGSetupTaskWaitPoint

```cpp
enum class ERDGSetupTaskWaitPoint : uint8 {
    Compile,   // コンパイル前に待機（デフォルト）
    Execute,   // 実行直前まで待機
};
```

---

## ERDGViewableResourceType

```cpp
enum class ERDGViewableResourceType : uint8 {
    Texture,
    Buffer,
    MAX,
};
```

---

## ERDGViewType

```cpp
enum class ERDGViewType : uint8 {
    TextureUAV,
    TextureSRV,
    BufferUAV,
    BufferSRV,
    MAX,
};
```

---

## ERDGUnorderedAccessViewFlags

```cpp
enum class ERDGUnorderedAccessViewFlags : uint8 {
    None            = 0,
    SkipBarrier     = 1 << 0,  // バリアをスキップ（同一パス内の複数 UAV 書き込み）
};
```

---

## ラムダ引数型と対応するコマンドリスト型

| ラムダ引数 | パスの種類 | 実行モード |
|-----------|----------|----------|
| `FRHICommandListImmediate&` | Raster / Compute / Copy | Inline |
| `FRHICommandList&` | Raster / Compute / Copy | Await |
| `FRHIComputeCommandList&` | Compute / AsyncCompute | Await |
| `FRDGAsyncTask, FRHICommandList&` | いずれも | Async |
| `FRDGDispatchPassBuilder&` | AddDispatchPass | — |
