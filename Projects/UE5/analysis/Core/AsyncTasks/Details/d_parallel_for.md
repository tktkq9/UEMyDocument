# ParallelFor — 並列ループ処理

- 上位: [[AsyncTasks/01_overview]]
- 関連: [[b_async_patterns]] | [[a_task_graph]]
- ソース: `Core/Public/Async/ParallelFor.h`, `Core/Public/Async/Async.h`

---

## 概要

`ParallelFor` は配列やインデックス範囲を複数スレッドで並列処理する最もシンプルな API。内部は `UE::Tasks::Launch` ベースで分割実行し、呼び出し元スレッドも参加してブロッキング完了する。

---

## 基本 API

```cpp
#include "Async/ParallelFor.h"

TArray<FVector> Positions;  // 入力配列（10000 要素など）
TArray<float>   Results;
Results.SetNum(Positions.Num());

// シンプルな並列ループ
ParallelFor(Positions.Num(), [&](int32 Index)
{
    Results[Index] = Positions[Index].Size();
});
// ParallelFor が戻ったとき全要素の処理完了が保証される
```

---

## EParallelForFlags

```cpp
// フラグ付きバージョン
ParallelFor(Positions.Num(), [&](int32 Index)
{
    Results[Index] = HeavyCompute(Positions[Index]);
},
EParallelForFlags::None);
```

| フラグ | 値 | 説明 |
|--------|---|------|
| `None` | `0` | デフォルト（自動並列化） |
| `ForceSingleThread` | `1` | 常にシングルスレッドで実行（デバッグ用） |
| `Unbalanced` | `2` | バッチ分割を均等にしない（各タスクが重さ不均一な場合） |
| `BackgroundPriority` | `4` | 低優先度スレッドで実行 |
| `PumpRenderingThread` | `8` | 実行中に RenderThread コマンドをフラッシュ（GameThread 呼び出し時） |

---

## バッチサイズと分割

`ParallelFor` は内部でインデックス範囲をバッチに分割してタスクとして投入する。

```cpp
// バッチサイズ（MinBatchSize）を直接指定
// 要素数 / MaxNumWorkerThreads 以下にはならない
ParallelForWithPreWork(
    Positions.Num(),
    [&](int32 Index) { Results[Index] = Compute(Index); },  // 本処理
    []() { /* 事前処理（分割前に 1 回だけ実行）*/ },
    EParallelForFlags::None
);
```

分割数は `FTaskGraphInterface::Get().GetNumWorkerThreads()` を基準にエンジンが決定する。要素数が少ない場合はシングルスレッドにフォールバックする。

---

## ParallelForTemplate — 型付きバージョン

```cpp
// TArray<T> を直接渡すバージョン
TArray<FMeshData> MeshArray;

ParallelForTemplate(MeshArray, [](FMeshData& Mesh)
{
    Mesh.ComputeNormals();
});
```

---

## 書き込み競合の回避

並列ループ内で共有データを書くときは **インデックスが重複しない** 設計か、スレッドセーフな手段が必要:

```cpp
// OK: 各スレッドが別インデックスに書く（競合なし）
TArray<int32> OutputArray;
OutputArray.SetNum(N);
ParallelFor(N, [&](int32 i) { OutputArray[i] = i * 2; });

// OK: FThreadSafeCounter でアトミック集計
FThreadSafeCounter TotalCount;
ParallelFor(N, [&](int32 i)
{
    if (SomeCondition(i)) TotalCount.Increment();
});
int32 Result = TotalCount.GetValue();

// OK: FCriticalSection で書き込みを保護（競合が稀な場合）
FCriticalSection Mutex;
TArray<int32> FilteredResults;
ParallelFor(N, [&](int32 i)
{
    if (SomeCondition(i))
    {
        FScopeLock Lock(&Mutex);
        FilteredResults.Add(i);
    }
});

// NG: TArray への Add は非スレッドセーフ（クラッシュ）
// ParallelFor(N, [&](int32 i) { FilteredResults.Add(i); });  // 危険
```

---

## TAtomic — アトミック型

```cpp
#include "Templates/Atomic.h"

TAtomic<int32> AtomicCounter(0);

ParallelFor(N, [&](int32 i)
{
    AtomicCounter.fetch_add(1, std::memory_order_relaxed);
});

int32 Total = AtomicCounter.load();
```

`std::atomic<T>` と同等。UE 型の方が一部プラットフォームで最適化される。

---

## FRWLock — 読み書きロック

多数の読み取りと少数の書き込みがある場合:

```cpp
FRWLock RWLock;

// 読み取り（複数スレッドが同時にOK）
{
    FReadScopeLock ReadLock(RWLock);
    float Val = SharedData[Index];
}

// 書き込み（排他ロック）
{
    FWriteScopeLock WriteLock(RWLock);
    SharedData.Add(NewEntry);
}
```

---

## ParallelFor vs UE::Tasks 比較

| 項目 | `ParallelFor` | `UE::Tasks::Launch` |
|------|--------------|---------------------|
| 用途 | インデックス範囲の並列処理 | 任意タスクの並列実行 |
| 完了待機 | 呼び出し元でブロック（同期） | `.Wait()` / `.GetResult()` で制御 |
| 依存関係 | なし | Prerequisites で DAG 構築可 |
| 戻り値 | なし | `TTask<T>` で取得 |
| オーバーヘッド | 低（自動バッチ） | やや高（タスク単位） |
| 向いている処理 | 均一な大量要素処理 | 依存関係のある複合処理 |

---

## 実践パターン

### 分割処理 + 後処理

```cpp
// 並列計算 → 結果をまとめて後処理
TArray<float> PartialSums;
PartialSums.SetNum(FTaskGraphInterface::Get().GetNumWorkerThreads() + 1);

ParallelFor(N, [&](int32 i)
{
    // スレッドローカルインデックスを取得して書き込み
    // （完全なスレッドローカル実装は LowLevelTasks の WorkerThread コンテキストが必要）
    PartialSums[0] += Data[i];  // シンプル版：単一アキュムレータ＋Mutex
});
```

### FPlatformProcess::Sleep を避ける

```cpp
// NG: ビジーウェイトは CPU を無駄使い
while (!bDone) { FPlatformProcess::Sleep(0.001f); }

// OK: FEvent で効率よく待機
FEvent* DoneEvent = FPlatformProcess::GetSynchEventFromPool();
ParallelFor(N, [&, DoneEvent](int32 i) {
    DoWork(i);
    if (i == N - 1) DoneEvent->Trigger();
});
DoneEvent->Wait();
FPlatformProcess::ReturnSynchEventToPool(DoneEvent);
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `TaskGraph.NumWorkerThreads` | 自動 | ワーカースレッド数（0=自動決定） |
| `ParallelFor.bForceSingleThread` | `0` | 強制シングルスレッド化（デバッグ用） |

---

## 関連ドキュメント

- [[b_async_patterns]] — `UE::Tasks::Launch` との比較・FRunnable
- [[a_task_graph]] — `FTaskGraph` による依存関係制御
- [[Reference/ref_async_api]] — `ParallelFor` / `TAtomic` / `FRWLock` API
