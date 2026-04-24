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

## 関連ドキュメント

- [[a_task_graph]] — FTaskGraph・ENamedThreads（GameThread/RenderThread 指定）
- [[b_async_patterns]] — 汎用非同期パターン
- [[Reference/ref_async_api]] — ENQUEUE_RENDER_COMMAND / FRHICommandList API
