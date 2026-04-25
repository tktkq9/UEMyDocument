# GameThread / RenderThread / RHIThread 連携

- 上位: [[AsyncTasks/01_overview]]
- 関連: [[a_task_graph]] | [[b_async_patterns]]
- ソース: `RenderCore/Public/RenderingThread.h`, `RHI/Public/RHICommandList.h`, `Engine/Public/UnrealEngine.h`

---

## スレッドモデル概要

UE5 の主要スレッドは 3 つ:

```
GameThread
  │  ENQUEUE_RENDER_COMMAND / EnqueueUniqueRenderCommand
  ▼
RenderThread（描画コマンド構築）
  │  RHICommandList への書き込み
  ▼
RHIThread（RHIコマンド実行・GPU発行）
  │
  ▼
GPU
```

| スレッド | 役割 | 主要関数 |
|---------|------|---------|
| GameThread | ゲームロジック・UObject 操作 | `IsInGameThread()` |
| RenderThread | シーン描画コマンドの構築 | `IsInRenderingThread()` |
| RHIThread | RHI コマンドの GPU 発行 | `IsInRHIThread()` |

---

## GameThread → RenderThread への投入

### ENQUEUE_RENDER_COMMAND（最も一般的）

```cpp
#include "RenderingThread.h"

// GameThread から RenderThread にコマンドを投入
ENQUEUE_RENDER_COMMAND(MyRenderCommand)(
    [MyData](FRHICommandListImmediate& RHICmdList)
    {
        // RenderThread 上で実行
        // RHICmdList を通じて GPU コマンドを積む
        RHICmdList.SetViewport(0, 0, 0, Width, Height, 1.0f);
    });
```

マクロ展開後は `FFunctionGraphTask::CreateAndDispatchWhenReady` に変換される。第一引数はデバッグ名として使われる。

### EnqueueUniqueRenderCommand（ラムダなし版）

```cpp
struct FMyRenderWork
{
    FMyRenderWork(float InVal) : Val(InVal) {}
    float Val;

    FORCEINLINE TStatId GetStatId() const
    {
        RETURN_QUICK_DECLARE_CYCLE_STAT(FMyRenderWork, STATGROUP_TaskGraphTasks);
    }

    static ENamedThreads::Type GetDesiredThread()
    {
        return ENamedThreads::RenderThread;
    }

    static ESubsequentsMode::Type GetSubsequentsMode()
    {
        return ESubsequentsMode::FireAndForget;
    }

    void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& Completion)
    {
        // RenderThread 上で実行
    }
};

TGraphTask<FMyRenderWork>::CreateTask().ConstructAndDispatchWhenReady(3.14f);
```

---

## RenderThread → GameThread への通知

`AsyncTask` / `FFunctionGraphTask` を使って GameThread に戻る:

```cpp
// RenderThread 内から GameThread に通知
AsyncTask(ENamedThreads::GameThread, []()
{
    // GameThread 上で実行
    // UObject にアクセスしてもよい
    GEngine->AddOnScreenDebugMessage(-1, 5.f, FColor::Green, TEXT("Done"));
});
```

---

## FlushRenderingCommands — 同期待機

GameThread から RenderThread のキューをすべてフラッシュして完了を待つ:

```cpp
// RenderThread のすべてのコマンドが完了するまで GameThread をブロック
// エディタ・ツール・レベル遷移時に使用。ゲームループ内での使用は非推奨
FlushRenderingCommands();
```

```cpp
// より細粒度の待機：特定の FGraphEventRef が完了するまで待つ
FRHICommandListExecutor::GetImmediateCommandList().ImmediateFlush(
    EImmediateFlushType::WaitForOutstandingTasksOnly);
```

---

## RHICommandList

RenderThread 上で GPU コマンドを構築するオブジェクト:

```cpp
// ENQUEUE_RENDER_COMMAND 内で受け取る
ENQUEUE_RENDER_COMMAND(DrawMesh)(
    [MeshData](FRHICommandListImmediate& RHICmdList)
    {
        // ビューポート設定
        RHICmdList.SetViewport(0, 0, 0.f, ViewWidth, ViewHeight, 1.f);

        // 頂点バッファバインド
        RHICmdList.SetStreamSource(0, VertexBuffer, 0);

        // 描画
        RHICmdList.DrawIndexedPrimitive(
            IndexBuffer, 0, 0, NumVertices, 0, NumTriangles, 1);
    });
```

### 主要メソッド

| メソッド | 説明 |
|---------|------|
| `SetViewport` | ビューポート設定 |
| `SetScissorRect` | シザー矩形 |
| `SetRenderTargets` | レンダーターゲット設定 |
| `CopyTexture` | テクスチャコピー |
| `DrawPrimitive` | 非インデックス描画 |
| `DrawIndexedPrimitive` | インデックス描画 |
| `DispatchComputeShader` | コンピュートシェーダ実行 |
| `Transition` | リソースバリア設定 |

---

## スレッド安全アクセスパターン

### データの受け渡しルール

| 操作 | 正しいスレッド |
|------|--------------|
| `UObject` の読み書き | GameThread のみ |
| RHI リソース作成/解放 | RenderThread or RHIThread |
| `FRenderResource` (FVertexBuffer 等) の InitRHI/ReleaseRHI | RenderThread |
| `GRenderTargetPool` アクセス | RenderThread |

### FThreadSafeCounter / FThreadSafeBool

```cpp
// カウンタ
FThreadSafeCounter Counter;
Counter.Increment();
int32 Val = Counter.GetValue();

// フラグ
FThreadSafeBool bReady;
bReady = true;
if (bReady) { /* ... */ }
```

### FCriticalSection（Mutex）

```cpp
FCriticalSection Mutex;

// ロック取得（スコープ外で自動解放）
{
    FScopeLock Lock(&Mutex);
    SharedData.Add(NewEntry);
}
```

### FEvent（シグナル待機）

```cpp
// イベント生成
FEvent* Event = FPlatformProcess::GetSynchEventFromPool(false);  // false=手動リセット

// 別スレッドからシグナル
Event->Trigger();

// 待機（タイムアウト付き）
bool bSignaled = Event->Wait(1000);  // 1000ms

// プールへ返却
FPlatformProcess::ReturnSynchEventToPool(Event);
```

---

## RenderThread のライフサイクル

```
StartRenderingThread()          ← エンジン起動時
  → FRenderingThread 作成（FRunnable 派生）
  → RenderingThread ID を記録（IsInRenderingThread() が使うスレッドID）

...フレームごとに ENQUEUE_RENDER_COMMAND...

FlushRenderingCommands()        ← レベル遷移・終了時
StopRenderingThread()           ← エンジン終了時
```

`GIsRenderingThreadSuspended` フラグで一時停止状態を確認できる。

---

## よくある間違い

```cpp
// NG: GameThread 以外での UObject アクセス
ENQUEUE_RENDER_COMMAND(Bad)(
    [Actor](FRHICommandListImmediate& RHICmdList)
    {
        Actor->SomeProp = 1;  // クラッシュ/競合の危険
    });

// OK: 値をコピーして渡す
float CopiedValue = Actor->SomeProp;
ENQUEUE_RENDER_COMMAND(Good)(
    [CopiedValue](FRHICommandListImmediate& RHICmdList)
    {
        // プリミティブ値ならOK
    });
```

---

## コード実行フロー

### エントリポイント（GT → RT → RHIThread → GPU）

```
(GameThread → RenderThread)
ENQUEUE_RENDER_COMMAND(Name)([Captured](FRHICommandListImmediate& RHI){...})  [RenderingThread.h]
  └─ TEnqueueUniqueRenderCommandType<Name>::EnqueueUniqueRenderCommand(Lambda)
       └─ FFunctionGraphTask::CreateAndDispatchWhenReady(
            Lambda, StatId, GameThreadCompletionEvent,
            ENamedThreads::GetRenderThread_Local())                  ← RT 専用キュー

(RenderThread 実行)
FRenderingThread::Run()                                            [RenderingThread.cpp]
  └─ FTaskGraphInterface::ProcessThreadUntilIdle(ENamedThreads::GetRenderThread())
       └─ ProcessTasks() ループ
            └─ Task->ExecuteTask() → Lambda(RHICmdList) 実行
                 └─ RHICmdList.SetRenderTargets/Draw/Dispatch       ← RHI コマンド積み

(RT → RHIThread)
RHICmdList.Flush() / RHIThread キック                              [RHICommandList.cpp]
  └─ FRHICommandListExecutor::ExecuteList(CmdList)
       ├─ if (RHIThread 有効) RHIThread に投入                       ← FRHITask
       └─ else GPU に直接発行（即時実行）

(RHIThread 実行)
FRHIThread::Run()                                                  [RHICommandList.cpp]
  └─ for each command: cmd->Execute(D3D12Context)
       └─ ID3D12CommandList::Dispatch / Draw                        ← GPU 命令発行

(RenderThread → GameThread)
AsyncTask(ENamedThreads::GameThread, []() { ... })                 [Async.h]
  └─ FFunctionGraphTask::CreateAndDispatchWhenReady(..., GameThread)
       └─ GameThread の Tick で TaskGraph キュー消化時に実行

(同期待機)
FlushRenderingCommands()                                           [RenderingThread.cpp]
  └─ FRHICommandListImmediate::ImmediateFlush(WaitForOutstandingTasks)
       └─ RT/RHI のすべてのタスクが完了するまで GameThread をブロック ← レベル遷移等
```

### フロー詳細

1. **マクロ展開** — `ENQUEUE_RENDER_COMMAND(Name)(Lambda)` は `TEnqueueUniqueRenderCommandType` 経由で `FFunctionGraphTask::CreateAndDispatchWhenReady` に展開され、`ENamedThreads::GetRenderThread_Local()` をターゲットに投入される（[[a_task_graph]]）。
2. **キャプチャの安全性** — Lambda がキャプチャする UObject ポインタは RT 上では触れない。プリミティブ値や RHI リソースのみキャプチャするのが鉄則。
3. **RenderThread ループ** — `FRenderingThread`（FRunnable）が `ProcessThreadUntilIdle` でキューを消化。各タスクが `RHICmdList` 経由で RHI コマンドを積む。
4. **RHIThread 分離** — RHI コマンドは `FRHICommandListExecutor` が `RHIThread` に転送。RT は次フレームの構築を続けられる（パイプライン化）。
5. **GPU 発行** — RHIThread が D3D12/Vulkan/Metal の Context API を呼んで GPU コマンドを発行。
6. **GT に戻る** — RT 内で GT 処理が必要なら `AsyncTask(ENamedThreads::GameThread, Lambda)` で TaskGraph 経由で戻す。
7. **同期点** — `FlushRenderingCommands()` は GT をブロックして RT/RHI のキューを空にする。レベル遷移・終了処理・スクリーンショット撮影で使用。ゲームループ中の使用は厳禁（フレーム時間が崩壊する）。
8. **同期プリミティブ** — `FCriticalSection`/`UE::FMutex` で共有データを保護。`FThreadSafeCounter`/`std::atomic` でカウンタ。`FEvent` でシグナル待機（[[b_async_patterns]]）。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `ENQUEUE_RENDER_COMMAND` | `RenderingThread.h` | GT → RT 投入マクロ |
| `TEnqueueUniqueRenderCommandType` | `RenderingThread.h` | マクロ展開先テンプレート |
| `FRenderingThread::Run` | `RenderingThread.cpp` | RT メインループ |
| `FRHICommandListExecutor::ExecuteList` | `RHICommandList.cpp` | RHIThread 投入 |
| `FRHIThread::Run` | `RHICommandList.cpp` | RHIThread メインループ |
| `FlushRenderingCommands` | `RenderingThread.cpp` | RT 完全フラッシュ |
| `AsyncTask(ENamedThreads::GameThread, ...)` | `Async.h` | RT → GT 通知 |
| `IsInGameThread` / `IsInRenderingThread` | `RenderingThread.h` | スレッド ID チェック |

---

## 関連ドキュメント

- [[a_task_graph]] — FTaskGraph・ENamedThreads（GameThread/RenderThread 指定）
- [[b_async_patterns]] — 汎用非同期パターン
- [[Reference/ref_async_api]] — ENQUEUE_RENDER_COMMAND / FRHICommandList API
