# TaskGraph — タスクグラフシステム

- 上位: [[AsyncTasks/01_overview]]
- 関連: [[b_async_patterns]] | [[c_game_thread]]
- ソース: `Core/Public/Async/TaskGraphInterfaces.h`

---

## 概要

`FTaskGraph` は UE4 から続く **DAG（有向非巡回グラフ）ベースのタスクスケジューラ**。タスク間の前提条件（Prerequisite）を指定することで依存関係を宣言的に記述できる。GameThread / RenderThread / 任意ワーカースレッドへの投入を `ENamedThreads` で制御し、複雑な非同期フローを構築できる。

---

## 基本 API

### FFunctionGraphTask — 最も簡単な使い方

```cpp
#include "Async/TaskGraphInterfaces.h"

// タスクを生成して即ディスパッチ
FGraphEventRef TaskEvent = FFunctionGraphTask::CreateAndDispatchWhenReady(
    []()
    {
        // 任意スレッドで実行
        DoWork();
    },
    TStatId(),                                     // 統計用ID（TStatId() でOK）
    nullptr,                                       // 前提条件（後述）
    ENamedThreads::AnyBackgroundThreadNormalTask   // 実行先スレッド
);

// 完了まで待機
FTaskGraphInterface::Get().WaitUntilTaskCompletes(TaskEvent);
```

### 特定スレッドで実行

```cpp
// GameThread で実行
FFunctionGraphTask::CreateAndDispatchWhenReady(
    []() { /* GameThread 上の処理 */ },
    TStatId(), nullptr,
    ENamedThreads::GameThread
);

// RenderThread で実行
FFunctionGraphTask::CreateAndDispatchWhenReady(
    []() { /* RenderThread 上の処理 */ },
    TStatId(), nullptr,
    ENamedThreads::RenderThread
);
```

---

## ENamedThreads — 実行先スレッド

| 値 | 説明 |
|----|------|
| `GameThread` | ゲームループスレッド |
| `RenderThread` | 描画コマンド構築スレッド |
| `AnyThread` | 任意のワーカー（優先度：Normal） |
| `AnyBackgroundThreadNormalTask` | バックグラウンドワーカー（優先度：Normal） |
| `AnyHiPriThreadHiPriTask` | 高優先度ワーカー |
| `AnyNormalThreadNormalTask` | 通常優先度ワーカー |
| `AnyNormalThreadHiPriTask` | 通常スレッド・高優先タスク |

`AnyThread` vs `AnyBackgroundThreadNormalTask`:
- `AnyThread` は**ゲームフレームに影響しうる**ワーカーも含む
- `AnyBackgroundThreadNormalTask` は GameThread を圧迫しない低優先度プールに限定

---

## 前提条件（Prerequisites）

複数タスクが完了してから実行する依存関係:

```cpp
// タスク A と B を生成
FGraphEventRef TaskA = FFunctionGraphTask::CreateAndDispatchWhenReady(
    []() { PrepareData(); }, TStatId(), nullptr, ENamedThreads::AnyThread);

FGraphEventRef TaskB = FFunctionGraphTask::CreateAndDispatchWhenReady(
    []() { PrepareOtherData(); }, TStatId(), nullptr, ENamedThreads::AnyThread);

// A と B 両方完了後に実行するタスク C
FGraphEventArray Prerequisites = { TaskA, TaskB };
FGraphEventRef TaskC = FFunctionGraphTask::CreateAndDispatchWhenReady(
    []() { ProcessResults(); },
    TStatId(),
    &Prerequisites,
    ENamedThreads::GameThread
);
```

---

## FGraphEventRef — タスクイベントハンドル

```cpp
// FGraphEventRef は TRefCountPtr<FGraphEvent> の typedef
FGraphEventRef Event = FFunctionGraphTask::CreateAndDispatchWhenReady(...);

// 配列
FGraphEventArray Events;
Events.Add(Event1);
Events.Add(Event2);

// 全タスク完了を待つ
FTaskGraphInterface::Get().WaitUntilTasksComplete(Events);

// 完了チェック（ポーリング）
bool bDone = Event->IsComplete();
```

---

## FBaseGraphTask — カスタムタスクの実装

より制御が必要な場合は `TGraphTask` テンプレートを使う:

```cpp
class FMyTask
{
public:
    FMyTask(int32 InData) : Data(InData) {}

    // 実行先スレッド（静的）
    static ENamedThreads::Type GetDesiredThread()
    {
        return ENamedThreads::AnyBackgroundThreadNormalTask;
    }

    // このタスクが後継タスクを持てるか
    static ESubsequentsMode::Type GetSubsequentsMode()
    {
        return ESubsequentsMode::TrackSubsequents;
    }

    // TStatId
    FORCEINLINE TStatId GetStatId() const
    {
        RETURN_QUICK_DECLARE_CYCLE_STAT(FMyTask, STATGROUP_TaskGraphTasks);
    }

    // タスク本体
    void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
    {
        // 処理
        ProcessData(Data);
    }

private:
    int32 Data;
};

// 生成と投入
FGraphEventRef TaskEvent = TGraphTask<FMyTask>::CreateTask(
    nullptr,   // Prerequisites
    ENamedThreads::GameThread
).ConstructAndDispatchWhenReady(42);  // FMyTask のコンストラクタ引数
```

---

## FTaskGraphInterface::Get() — グローバルアクセス

```cpp
// インスタンス取得
FTaskGraphInterface& TGI = FTaskGraphInterface::Get();

// タスク完了待機
TGI.WaitUntilTaskCompletes(Event);
TGI.WaitUntilTasksComplete(Events);

// 現在のスレッドを取得
ENamedThreads::Type CurrentThread = TGI.GetCurrentThreadIfKnown();

// ワーカースレッド数
int32 NumWorkers = TGI.GetNumWorkerThreads();
```

---

## TaskGraph vs Task System v2（比較）

| 項目 | FTaskGraph（レガシー） | UE::Tasks（v2） |
|------|---------------------|----------------|
| API | `FFunctionGraphTask` | `UE::Tasks::Launch` |
| 依存関係 | `FGraphEventArray` | `Prerequisites(TaskRef)` |
| 待機 | `WaitUntilTaskCompletes` | `.Wait()` |
| スレッド指定 | `ENamedThreads` | `ETaskPriority` |
| オーバーヘッド | 大（タスクノード割り当て） | 小（スタック最適化） |
| ワークスティーリング | No | Yes |

新規コードには `UE::Tasks::Launch`（Task System v2）が推奨。既存コードとの混在も可能。

---

## コード実行フロー

### エントリポイント（タスク投入 〜 前提条件解決 〜 実行）

```
(タスク生成)
FFunctionGraphTask::CreateAndDispatchWhenReady(Lambda, StatId, Prereq, Thread)
                                                                  [TaskGraphInterfaces.h]
  └─ TGraphTask<FFunctionGraphTask>::CreateTask(Prereq, CurrentThread)
       └─ FConstructor::ConstructAndDispatchWhenReady(Lambda)
            ├─ new FFunctionGraphTask(Lambda) (placement new)
            ├─ FBaseGraphTask::Construct                            ← TaskGraph に登録
            └─ Schedule()
                 └─ if (Prereq 全完了) Queue() 即実行待ち列へ
                 └─ else PrereqDone コールバック登録                  ← 完了通知を待つ

(実行ディスパッチ)
FBaseGraphTask::Schedule()                                         [TaskGraph.cpp]
  └─ FTaskGraphInterface::QueueTask(Task, ThreadToExecuteOn)
       ├─ if (Named Thread) FNamedTaskThread::EnqueueFromThisThread() ← 専用キュー
       └─ if (AnyThread)    FTaskThreadAnyThread::EnqueueFromOtherThread() ← ワーカープール

(ワーカー実行)
FTaskThreadAnyThread::ProcessTasks()                               [TaskGraph.cpp]
  └─ while (true):
       ├─ Task = Queue.Pop()
       └─ Task->ExecuteTask(NewTasks, CurrentThread)
            └─ FFunctionGraphTask::DoTask(CurrentThread, MyCompletionEvent)
                 └─ Function(CurrentThread, MyCompletionEvent)      ← Lambda 実行

(完了通知)
Task->ConditionalQueueAnotherTask()
  └─ for each subsequent task:
       └─ if (PrereqsRemaining == 0) Schedule() で次タスク起動    ← DAG 進行

(待機)
FTaskGraphInterface::WaitUntilTaskCompletes(Event, ENamedThreads)
  └─ if (IsInGameThread()) FNamedTaskThread::ProcessTasksUntilQuit()
       └─ 待機中も他タスクを処理して進行                           ← デッドロック回避
```

### フロー詳細

1. **タスクオブジェクト生成** — `TGraphTask<T>::CreateTask` が `FBaseGraphTask` 派生のタスクをアロケータ経由で生成し、TaskGraph に登録する。
2. **Prerequisites 評価** — `Schedule` 時点で前提条件 `FGraphEventRef` の完了状態をチェック。未完了なら各 prereq の subsequent リストに自身を追加し、完了通知を待つ。
3. **ディスパッチ** — 全 prereq が完了済みなら `FTaskGraphInterface::QueueTask` で実行先スレッドのキューに投入。`ENamedThreads` に応じて `FNamedTaskThread`（GameThread/RenderThread 専用）または `FTaskThreadAnyThread`（ワーカープール）が拾う。
4. **タスク実行** — ワーカースレッドが `Task->ExecuteTask` を呼ぶ。`FFunctionGraphTask::DoTask` が登録された Lambda を実行する。
5. **DAG 進行** — タスク完了時、subsequent タスクの prereq 残数を減らし、0 になったタスクを再ディスパッチ。これで DAG が自動進行する。
6. **待機戦略** — `WaitUntilTaskCompletes` は GameThread から呼んだ場合、待機中も自分のキューにある他タスクを処理することでデッドロックを避ける（[[c_game_thread]]）。
7. **TGraphTask カスタム** — `GetDesiredThread`・`GetSubsequentsMode`・`DoTask` を持つクラスを `TGraphTask<T>::CreateTask` に渡せばカスタムタスクが作れる。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `FFunctionGraphTask::CreateAndDispatchWhenReady` | `TaskGraphInterfaces.h` | Lambda タスクの簡易投入 |
| `TGraphTask<T>::CreateTask` | `TaskGraphInterfaces.h` | カスタムタスク生成 |
| `FBaseGraphTask::Schedule` / `ConditionalQueueAnotherTask` | `TaskGraph.cpp` | ディスパッチ・DAG 進行 |
| `FTaskGraphInterface::QueueTask` | `TaskGraph.cpp` | スレッド振分 |
| `FNamedTaskThread::ProcessTasksUntilQuit` | `TaskGraph.cpp` | GameThread/RenderThread 実行ループ |
| `FTaskThreadAnyThread::ProcessTasks` | `TaskGraph.cpp` | ワーカースレッド実行ループ |
| `FGraphEvent::IsComplete` / `Wait` | `TaskGraphInterfaces.h` | 完了問合せ・待機 |

---

## 関連ドキュメント

- [[b_async_patterns]] — `Async()` / `UE::Tasks::Launch` / `FRunnable` との比較
- [[c_game_thread]] — GameThread / RenderThread 間の連携
- [[Reference/ref_async_api]] — `FTaskGraphInterface` / `FFunctionGraphTask` API
