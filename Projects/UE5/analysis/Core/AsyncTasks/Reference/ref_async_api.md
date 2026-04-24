# Async / FAsyncTask / FRunnable API リファレンス

- 上位: [[AsyncTasks/01_overview]]
- 関連: [[Details/a_task_graph]] | [[Details/b_async_patterns]] | [[Details/c_game_thread]] | [[Details/d_parallel_for]]
- ソース: `Core/Public/Async/Async.h`, `Core/Public/Async/AsyncWork.h`, `Core/Public/HAL/Runnable.h`, `Core/Public/Tasks/Task.h`

---

## Async() + TFuture

```cpp
#include "Async/Async.h"

// 単発非同期実行（戻り値あり）
TFuture<T> Async(EAsyncExecution Execution, TFunction<T()> Function, TFunction<void()> CompletionCallback = nullptr);

// EAsyncExecution
enum class EAsyncExecution
{
    TaskGraph,               // TaskGraph ワーカー（推奨）
    TaskGraphMainThread,     // GameThread（TaskGraph 経由）
    Thread,                  // 新規スレッド生成
    ThreadPool,              // GThreadPool からスレッドを借用
    LargeThreadPool,         // GLargeThreadPool
};

// 例
TFuture<int32> Future = Async(EAsyncExecution::TaskGraph, []() -> int32
{
    return HeavyCompute();
});
int32 Result = Future.Get();  // ブロッキング待機
```

### TFuture / TPromise

```cpp
// Promise で Future を制御
TPromise<FString> Promise;
TFuture<FString> Future = Promise.GetFuture();

Promise.SetValue(TEXT("Done"));   // 別スレッドから値をセット
FString Val = Future.Get();       // 完了まで待機

// コールバック（非ブロッキング）
Future.Then([](TFuture<FString> F) {
    FString Result = F.Get();
});

// TFuture<void> — 戻り値なし版
TFuture<void> VoidFuture = Async(EAsyncExecution::Thread, []() { DoWork(); });
VoidFuture.Wait();
```

---

## UE::Tasks — Task System v2

```cpp
#include "Tasks/Task.h"

// 起動
UE::Tasks::FTask UE::Tasks::Launch(
    const TCHAR* DebugName,
    TFunction<void()> TaskBody,
    UE::Tasks::ETaskPriority Priority = UE::Tasks::ETaskPriority::Normal,
    UE::Tasks::EExtendedTaskPriority ExtendedPriority = UE::Tasks::EExtendedTaskPriority::None
);

// 戻り値付き
UE::Tasks::TTask<T> UE::Tasks::Launch(
    const TCHAR* DebugName,
    TFunction<T()> TaskBody,
    ...
);

// 待機
void FTask::Wait();                // 完了まで現スレッドをブロック
void FTask::BusyWait();           // スピン待機

// 依存関係
auto Prerequisites = UE::Tasks::Prerequisites(TaskA, TaskB);
UE::Tasks::FTask TaskC = UE::Tasks::Launch(TEXT("C"), []() {}, Prerequisites);

// 結果取得
T FTask::GetResult();  // TTask<T> のみ

// 優先度
enum class ETaskPriority { High, Normal, BackgroundHigh, BackgroundNormal, BackgroundLow };
```

---

## FFunctionGraphTask（TaskGraph v1）

```cpp
#include "Async/TaskGraphInterfaces.h"

// 起動
FGraphEventRef FFunctionGraphTask::CreateAndDispatchWhenReady(
    TFunction<void()> Function,
    TStatId         StatId,
    FGraphEventArray* Prerequisites,   // nullptr で依存なし
    ENamedThreads::Type DesiredThread  // 実行先スレッド
);

// 待機
FTaskGraphInterface::Get().WaitUntilTaskCompletes(Event);
FTaskGraphInterface::Get().WaitUntilTasksComplete(Events);

// スレッドポーリング
bool FGraphEvent::IsComplete() const;

// ENamedThreads
ENamedThreads::GameThread
ENamedThreads::RenderThread
ENamedThreads::AnyThread
ENamedThreads::AnyBackgroundThreadNormalTask
ENamedThreads::AnyHiPriThreadHiPriTask
ENamedThreads::AnyNormalThreadNormalTask

// TGraphTask（カスタムタスク）
FGraphEventRef TGraphTask<FMyTask>::CreateTask(&Prerequisites, ENamedThreads::GameThread)
    .ConstructAndDispatchWhenReady(ConstructorArgs...);
```

---

## FAsyncTask

```cpp
#include "Async/AsyncWork.h"

// タスク本体の定義
class FMyTask : public FNonAbandonableTask
{
    friend class FAsyncTask<FMyTask>;
    FMyTask(int32 In) : Input(In) {}

    void DoWork() { /* 重い処理 */ }

    FORCEINLINE TStatId GetStatId() const
    {
        RETURN_QUICK_DECLARE_CYCLE_STAT(FMyTask, STATGROUP_ThreadPoolAsyncTasks);
    }

    int32 Input;
};

// 実行
FAsyncTask<FMyTask>* Task = new FAsyncTask<FMyTask>(42);
Task->StartBackgroundTask();          // GThreadPool に投入
Task->StartSynchronousTask();         // 同期実行（テスト用）
Task->EnsureCompletion(bool bDoWorkOnThisThread=true);  // 完了まで待機
bool bDone = Task->IsDone();
delete Task;

// 自動削除版
(new FAutoDeleteAsyncTask<FMyTask>(42))->StartBackgroundTask();

// FAbandonableTask — 処理を中断できるタスク
class FMyAbandonableTask : public FNonAbandonableTask { /* ... */ };
// Abandon() で中断を通知できる（DoWork 内で検知する設計は自前）
```

---

## FRunnable / FRunnableThread

```cpp
#include "HAL/Runnable.h"
#include "HAL/RunnableThread.h"

// FRunnable の実装
class FMyRunnable : public FRunnable
{
    virtual bool   Init()     override;  // false で Run を呼ばずに終了
    virtual uint32 Run()      override;  // メインループ（必須）
    virtual void   Stop()     override;  // 停止要求（別スレッドから呼ばれる）
    virtual void   Exit()     override;  // Run() 終了後のクリーンアップ
};

// スレッド生成
FRunnableThread* Thread = FRunnableThread::Create(
    Runnable,            // FRunnable* インスタンス
    const TCHAR* Name,   // スレッド名
    uint32 StackSize = 0,               // 0 = デフォルト
    EThreadPriority Priority = TPri_Normal,
    uint64 AffinityMask = FPlatformAffinity::GetNoAffinityMask()
);

// 停止と待機
Thread->Suspend(bool bShouldPause);   // 一時停止
Thread->Kill(bool bShouldWait=false); // 強制終了
Thread->WaitForCompletion();          // Run() の完了を待つ
uint32 Thread->GetThreadID() const;

// EThreadPriority
TPri_TimeCritical, TPri_Highest, TPri_AboveNormal,
TPri_Normal, TPri_BelowNormal, TPri_Lowest, TPri_SlightlyBelowNormal
```

---

## ParallelFor

```cpp
#include "Async/ParallelFor.h"

// 基本形
void ParallelFor(int32 Num, TFunction<void(int32)> Body, EParallelForFlags Flags = EParallelForFlags::None);

// 事前処理付き
void ParallelForWithPreWork(int32 Num, TFunction<void(int32)> Body, TFunction<void()> PreWork, EParallelForFlags Flags = EParallelForFlags::None);

// EParallelForFlags
enum class EParallelForFlags
{
    None             = 0,
    ForceSingleThread = 1,   // デバッグ用シングルスレッド強制
    Unbalanced       = 2,    // バッチサイズを均等にしない
    BackgroundPriority = 4,  // 低優先度実行
    PumpRenderingThread = 8, // 実行中に RenderThread をフラッシュ
};
```

---

## スレッド同期プリミティブ

```cpp
// Mutex
FCriticalSection Mutex;
FScopeLock Lock(&Mutex);

// 読み書きロック
FRWLock RWLock;
FReadScopeLock ReadLock(RWLock);
FWriteScopeLock WriteLock(RWLock);

// イベント（シグナル）
FEvent* Event = FPlatformProcess::GetSynchEventFromPool(bool bIsManualReset);
Event->Trigger();
Event->Wait(uint32 WaitTimeMs = INFINITE);
Event->Reset();
FPlatformProcess::ReturnSynchEventToPool(Event);

// アトミック
FThreadSafeCounter Counter;
Counter.Increment();
Counter.Decrement();
int32 Val = Counter.GetValue();
Counter.Set(0);

FThreadSafeBool Flag;
Flag = true;
bool bVal = (bool)Flag;

TAtomic<T> AtomicVal;
AtomicVal.fetch_add(1);
AtomicVal.store(0);
T V = AtomicVal.load();
```

---

## ENQUEUE_RENDER_COMMAND

```cpp
#include "RenderingThread.h"

// GameThread → RenderThread への投入
ENQUEUE_RENDER_COMMAND(CommandName)(
    [CapturedData](FRHICommandListImmediate& RHICmdList)
    {
        // RenderThread 上で実行
    });

// RenderThread → GameThread
AsyncTask(ENamedThreads::GameThread, []() { /* GameThread で実行 */ });

// RenderThread の完了を同期待ち（ゲームループ内では非推奨）
FlushRenderingCommands();
```

---

## 関連ドキュメント

- [[Details/a_task_graph]] — TaskGraph の概念・ENamedThreads
- [[Details/b_async_patterns]] — パターン選択ガイド
- [[Details/c_game_thread]] — RenderThread 連携の詳細
- [[Details/d_parallel_for]] — ParallelFor の詳細・同期プリミティブ
