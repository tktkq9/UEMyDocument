# 非同期パターン（Async / FAsyncTask / FRunnable / UE::Tasks）

- 上位: [[AsyncTasks/01_overview]]
- 関連: [[a_task_graph]] | [[c_game_thread]] | [[d_parallel_for]]
- ソース: `Core/Public/Async/Async.h`, `Core/Public/Async/AsyncWork.h`, `Core/Public/HAL/Runnable.h`, `Core/Public/Async/Fundamental/Task.h`

---

## 概要

UE5 の非同期処理には用途に応じて複数のパターンがある。どれを使うかは **タスクの寿命・スレッドモデル・依存関係の有無** で選ぶ。

| パターン | 向いているケース |
|---------|----------------|
| `Async()` / `TFuture` | シンプルな単発非同期処理 + 結果取得 |
| `UE::Tasks::Launch` | 軽量高頻度タスク（v2 推奨） |
| `FTaskGraph` | 複雑な依存関係を持つタスク DAG |
| `FAsyncTask<T>` | キュー方式の重いバックグラウンド処理 |
| `FRunnable` / `FRunnableThread` | ゲーム中ずっと動く長寿命スレッド |
| `ParallelFor` | 配列の並列処理 |

---

## Async() + TFuture

`std::async` 相当の最もシンプルな非同期実行:

```cpp
#include "Async/Async.h"

// バックグラウンドで実行し TFuture で結果を受け取る
TFuture<int32> Future = Async(EAsyncExecution::Thread, []() -> int32 {
    return HeavyComputation();
});

// 他の処理…

int32 Result = Future.Get();  // 完了まで待機してから取得
```

### EAsyncExecution

| 値 | 実行先 |
|----|--------|
| `TaskGraph` | TaskGraph ワーカー（推奨：GameThread を圧迫しない） |
| `TaskGraphMainThread` | GameThread（TaskGraph 経由） |
| `Thread` | 新規スレッド生成（単発用。スレッド生成コストあり） |
| `ThreadPool` | `GThreadPool` からスレッドを借用 |

### TFuture / TPromise

```cpp
// Promise を作って Future を渡す
TPromise<FString> Promise;
TFuture<FString> Future = Promise.GetFuture();

// 別スレッドから値を設定
Async(EAsyncExecution::Thread, [P = MoveTemp(Promise)]() mutable {
    P.SetValue(TEXT("Done"));
});

// GameThread で取得
FString Result = Future.Get();

// コールバック方式（ブロッキングなし）
Future.Then([](TFuture<FString> F) {
    FString Val = F.Get();
    // 完了時処理
});
```

---

## UE::Tasks — Task System v2（推奨）

UE5 新世代タスクシステム。FTaskGraph より低オーバーヘッド:

```cpp
#include "Tasks/Task.h"

// シンプルな起動
UE::Tasks::FTask Task = UE::Tasks::Launch(
    TEXT("MyTask"),            // デバッグ名
    []() { DoWork(); },
    UE::Tasks::ETaskPriority::Normal
);

Task.Wait();                   // 完了まで待機
// または
Task.BusyWait();               // スピン待機（GameThread など即時完了想定の場合）
```

### Prerequisites（依存関係）

```cpp
UE::Tasks::FTask A = UE::Tasks::Launch(TEXT("TaskA"), []() { PrepareA(); });
UE::Tasks::FTask B = UE::Tasks::Launch(TEXT("TaskB"), []() { PrepareB(); });

// A と B が完了してから実行
UE::Tasks::FTask C = UE::Tasks::Launch(
    TEXT("TaskC"),
    []() { ProcessAB(); },
    UE::Tasks::Prerequisites(A, B)
);
```

### 戻り値付きタスク

```cpp
UE::Tasks::TTask<int32> Task = UE::Tasks::Launch(
    TEXT("ComputeTask"),
    []() -> int32 { return 42; }
);

int32 Result = Task.GetResult();  // 完了まで待機して結果取得
```

---

## FAsyncTask\<T\> — キュー方式

重いバックグラウンド処理に向く。`FQueuedThreadPool` からスレッドを借用:

```cpp
// 1. タスク本体を定義
class FMyHeavyTask : public FNonAbandonableTask
{
    friend class FAsyncTask<FMyHeavyTask>;

    int32 InputData;

    FMyHeavyTask(int32 InData) : InputData(InData) {}

    // 重い処理
    void DoWork()
    {
        ProcessHeavily(InputData);
    }

    // タスク名（統計用）
    FORCEINLINE TStatId GetStatId() const
    {
        RETURN_QUICK_DECLARE_CYCLE_STAT(FMyHeavyTask, STATGROUP_ThreadPoolAsyncTasks);
    }
};

// 2. 非同期実行
FAsyncTask<FMyHeavyTask>* Task = new FAsyncTask<FMyHeavyTask>(42);
Task->StartBackgroundTask();    // GThreadPool に投入

// 3. 完了確認
if (Task->IsDone())
{
    delete Task;
}

// または同期完了待機
Task->EnsureCompletion();
delete Task;
```

`FAutoDeleteAsyncTask<T>` を使うと完了時に自動 `delete`:

```cpp
(new FAutoDeleteAsyncTask<FMyHeavyTask>(42))->StartBackgroundTask();
```

---

## FRunnable — 長寿命スレッド

ゲーム中ずっと動き続けるワーカースレッド（ネットワーク受信・ファイル監視等）:

```cpp
class FMyWorkerThread : public FRunnable
{
public:
    FMyWorkerThread() : bRunning(true) {}

    // スレッド開始時（スレッド上で呼ばれる）
    virtual bool Init() override
    {
        return true;  // false を返すと Run() を呼ばずに終了
    }

    // メインループ
    virtual uint32 Run() override
    {
        while (bRunning)
        {
            ProcessQueue();
            FPlatformProcess::Sleep(0.001f);  // 1ms スリープ
        }
        return 0;
    }

    // 停止要求（別スレッドから呼ばれる）
    virtual void Stop() override
    {
        bRunning = false;
    }

    // スレッド終了後（スレッド上で呼ばれる）
    virtual void Exit() override
    {
        CleanupResources();
    }

private:
    FThreadSafeBool bRunning;
};

// 使用
FMyWorkerThread* Worker = new FMyWorkerThread();
FRunnableThread* Thread = FRunnableThread::Create(
    Worker,
    TEXT("MyWorker"),
    0,                                          // スタックサイズ（0=デフォルト）
    TPri_BelowNormal                            // スレッド優先度
);

// 停止
Worker->Stop();
Thread->WaitForCompletion();
delete Thread;
delete Worker;
```

---

## スレッド優先度（EThreadPriority）

| 値 | 説明 |
|----|------|
| `TPri_TimeCritical` | リアルタイム（OSの最高優先度。慎重に使う）|
| `TPri_Highest` | 最高優先 |
| `TPri_AboveNormal` | 高優先 |
| `TPri_Normal` | 通常 |
| `TPri_BelowNormal` | 低優先（バックグラウンド処理向け）|
| `TPri_Lowest` | 最低 |
| `TPri_SlightlyBelowNormal` | 僅かに低い |

---

## パターン選択ガイド

```
短命・単発の処理
  ├─ 結果が必要 → Async() + TFuture
  ├─ 結果不要・軽量 → UE::Tasks::Launch
  ├─ 依存関係あり → UE::Tasks::Launch + Prerequisites
  └─ 重いバッチ処理 → FAsyncTask<T>

長命・継続処理
  └─ FRunnable + FRunnableThread

並列ループ
  └─ ParallelFor（[[d_parallel_for]] 参照）

GameThread / RenderThread 連携
  └─ ENQUEUE_RENDER_COMMAND（[[c_game_thread]] 参照）
```

---

## コード実行フロー

### エントリポイント（Async / UE::Tasks / FAsyncTask / FRunnable）

```
(Async + TFuture)
Async(EAsyncExecution::Thread, Lambda)                            [Async/Async.h]
  └─ TPromise<T> Promise; TFuture<T> Future = Promise.GetFuture();
       └─ switch (Execution):
            ├─ TaskGraph     → FFunctionGraphTask::CreateAndDispatchWhenReady
            ├─ TaskGraphMain → ENamedThreads::GameThread
            ├─ Thread        → FRunnableThread::Create で単発スレッド
            └─ ThreadPool    → GThreadPool->AddQueuedWork(FAutoDeleteAsyncTask)
       └─ ラムダ完了時 Promise.SetValue(Result)                    ← Future 解決

(UE::Tasks v2)
UE::Tasks::Launch(DebugName, Lambda, Priority)                    [Tasks/Task.h]
  └─ FTaskBase* Task = new FTaskBase(...)
       └─ LowLevelTasks::FScheduler::Get().TryLaunch(Task)         [Async/Fundamental/Scheduler.cpp]
            └─ TLocalQueue / GlobalQueue にキュー
                 └─ ParkingLot で休眠ワーカーを起こす                ← futex/event
                      └─ ワーカーが Task->ExecuteTask() 実行
                           └─ ワークスティーリング: 自キュー枯渇時に他キューから盗む

(FAsyncTask - キュー方式)
Task = new FAsyncTask<FMyHeavyTask>(Args);                         [Async/AsyncWork.h]
Task->StartBackgroundTask()
  └─ GThreadPool->AddQueuedWork(this)                              [HAL/QueuedThreadPool.h]
       └─ FQueuedThread が AcceptWork() でタスクを引き取り
            └─ Task->DoThreadedWork()
                 └─ MyHeavyTask::DoWork()                          ← 実装本体

(FRunnable - 長寿命)
Worker = new FMyWorkerThread();
Thread = FRunnableThread::Create(Worker, "MyWorker", StackSize, Priority)  [HAL/RunnableThread.h]
  └─ FRunnableThreadWin (Win32 実装) が ::CreateThread() で OS スレッド生成
       └─ ThreadProc:
            ├─ Runnable->Init()
            ├─ Runnable->Run()                                      ← whileループ
            └─ Runnable->Exit()
Worker->Stop() → bRunning = false → Run() ループ終了
Thread->WaitForCompletion() → スレッドの終了待ち
```

### フロー詳細

1. **Async 経路** — `Async()` は `TPromise/TFuture` を生成し、選択した実行モード（`TaskGraph` / `Thread` / `ThreadPool` 等）に Lambda を投入。完了時に `Promise.SetValue` で `Future.Get()` がアンブロック。
2. **TFuture::Then** — コールバック方式で待機するなら `Future.Then(Lambda)`。完了時に Lambda が呼ばれる（ノンブロッキング）。
3. **UE::Tasks v2** — `LowLevelTasks::FScheduler` が ParkingLot ベースで管理。各ワーカーは `TLocalQueue` を持ち、空になると他ワーカーの GlobalQueue から盗む（ワークスティーリング、[[a_task_graph]] と低レイヤー機構を共有）。
4. **Prerequisites v2** — `UE::Tasks::Prerequisites(A, B)` で前提条件を指定すると、prereq の subsequent リストにこの task が登録される。前提完了時に再キュー。
5. **FAsyncTask** — `FQueuedThreadPool` から `FQueuedThread` を借用してタスクを実行。プールサイズは `GThreadPool->Create()` で決まる。`FAutoDeleteAsyncTask` は完了時に自動 delete。
6. **FRunnable** — `FRunnableThread::Create` がプラットフォーム固有実装（Win 系は `FRunnableThreadWin`）で OS スレッドを生成。`Init → Run → Exit` のライフサイクルでループ。
7. **スレッド優先度** — `EThreadPriority` を OS のスレッド優先度に変換。`TPri_TimeCritical` は OS 最高優先度のため、ゲームスレッドを圧迫する可能性に注意。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `Async` | `Async/Async.h` | 汎用非同期エントリ |
| `TFuture<T>::Get` / `Then` | `Async/Future.h` | 結果取得・コールバック |
| `UE::Tasks::Launch` | `Tasks/Task.h` | Task System v2 起動 |
| `LowLevelTasks::FScheduler::TryLaunch` | `Async/Fundamental/Scheduler.cpp` | v2 スケジューラ |
| `FAsyncTask<T>::StartBackgroundTask` | `Async/AsyncWork.h` | スレッドプール投入 |
| `FQueuedThreadPool::AddQueuedWork` | `HAL/QueuedThreadPool.h` | キュー方式投入 |
| `FRunnableThread::Create` | `HAL/RunnableThread.h` | 独立スレッド生成 |
| `FRunnableThreadWin::ThreadProc` | `Windows/WindowsRunnableThread.cpp` | Win32 スレッドプロシージャ |

---

## 関連ドキュメント

- [[a_task_graph]] — FTaskGraph の前提条件・ENamedThreads
- [[c_game_thread]] — GameThread/RenderThread 間のデータ受け渡し
- [[d_parallel_for]] — 配列の並列処理
- [[Reference/ref_async_api]] — `Async` / `FAsyncTask` / `FRunnable` の API
