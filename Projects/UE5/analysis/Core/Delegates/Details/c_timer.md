# タイマー（FTimerManager）

- 上位: [[Delegates/01_overview]]
- 関連: [[a_delegate_types]] | [[b_dynamic_delegates]]
- ソース: `Engine/Public/TimerManager.h`

---

## 概要

`FTimerManager` は GameThread 上でタイマーを管理するシステム。`UWorld::GetTimerManager()` または `GetWorldTimerManager()` でアクセスし、一定時間後・繰り返しの関数呼び出しをスケジュールできる。内部はデリゲート（`FTimerDelegate`）ベース。

---

## 基本 API

### SetTimer

```cpp
#include "Engine/World.h"  // GetWorldTimerManager()

FTimerHandle Handle;

// 2 秒後に 1 回だけ呼び出し
GetWorldTimerManager().SetTimer(
    Handle,
    this,
    &AMyActor::OnTimerFired,   // UFUNCTION 不要（メソッドポインタ）
    2.0f,                       // 遅延（秒）
    false                       // ループするか
);

// 1 秒ごとにループ呼び出し
GetWorldTimerManager().SetTimer(
    Handle,
    this,
    &AMyActor::OnTick,
    1.0f,
    true    // bLoop = true
);

// ラムダ版（FTimerDelegate で包む）
FTimerDelegate Delegate = FTimerDelegate::CreateLambda([this]()
{
    DoSomething();
});
GetWorldTimerManager().SetTimer(Handle, Delegate, 0.5f, false);
```

### FTimerHandle

```cpp
// ハンドルはタイマーの識別子
FTimerHandle MyTimerHandle;

// 有効かチェック
bool bActive = GetWorldTimerManager().IsTimerActive(MyTimerHandle);

// 残り時間取得
float Remaining = GetWorldTimerManager().GetTimerRemaining(MyTimerHandle);

// 経過時間取得
float Elapsed = GetWorldTimerManager().GetTimerElapsed(MyTimerHandle);

// 一時停止
GetWorldTimerManager().PauseTimer(MyTimerHandle);
GetWorldTimerManager().UnPauseTimer(MyTimerHandle);

// 停止・クリア
GetWorldTimerManager().ClearTimer(MyTimerHandle);
```

---

## SetTimerForNextTick

次フレームの開始時に 1 回だけ実行:

```cpp
// 次フレームで OnNextTick を呼ぶ
GetWorldTimerManager().SetTimerForNextTick(this, &AMyActor::OnNextTick);

// ラムダ版
GetWorldTimerManager().SetTimerForNextTick(FTimerDelegate::CreateLambda([this]()
{
    // 1 フレーム後に実行
    RefreshUI();
}));
```

ゲームフレームに同期した「遅延 0 秒実行」として使う。`AsyncTask(GameThread)` の代わりに使えることもある。

---

## タイマーの挙動

```
SetTimer(Handle, 2.0f, false)
  │
  2 秒後 → 関数呼び出し → タイマー終了

SetTimer(Handle, 1.0f, true)
  │
  1 秒後 → 関数呼び出し
  │
  2 秒後 → 関数呼び出し
  │
  ...（ClearTimer まで繰り返し）
```

- タイマーは **DeltaTime の積算**で進む。ゲームポーズ中は `FTimerManager` もポーズする（`bIgnorePause` フラグで制御可）
- `TimeScale` に従って速度が変わる（スローモーション対応）

---

## IgnorePause — ポーズ中も実行

```cpp
// ゲームがポーズ中でもタイマーを動かす
GetWorldTimerManager().SetTimer(
    Handle,
    FTimerDelegate::CreateUObject(this, &AMyActor::OnPausedTimer),
    1.0f,
    true,       // bLoop
    -1.0f,      // FirstDelay（-1 = Rate と同じ）
    true        // bIgnorePause
);
```

---

## Blueprint との対応

Blueprint の「Set Timer by Function Name」ノード = C++ の `SetTimerByFunctionName`:

```cpp
// UFUNCTION が必要
UFUNCTION()
void MyTimerCallback();

// 文字列指定版（Blueprint の「Set Timer by Function Name」相当）
GetWorldTimerManager().SetTimerByFunctionName(
    this,
    FName(TEXT("MyTimerCallback")),
    2.0f,
    false
);
```

通常 C++ では関数ポインタ版（`SetTimer`）を優先する。

---

## FTimerManager のアクセス方法

```cpp
// AActor 内から（最も一般的）
GetWorldTimerManager().SetTimer(...);

// UWorld から直接
UWorld* World = GetWorld();
if (World)
{
    World->GetTimerManager().SetTimer(...);
}

// UGameplayStatics 経由（静的アクセス。Context オブジェクトが必要）
UGameplayStatics::GetPlayerController(GetWorld(), 0); // Context として使う例

// UActorComponent 内から
GetWorld()->GetTimerManager().SetTimer(...);
```

---

## ライフタイムと注意点

```cpp
// BeginPlay / EndPlay ペアで管理するパターン
virtual void BeginPlay() override
{
    Super::BeginPlay();
    GetWorldTimerManager().SetTimer(UpdateHandle, this, &AMyActor::Update, 0.1f, true);
}

virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override
{
    GetWorldTimerManager().ClearTimer(UpdateHandle);
    Super::EndPlay(EndPlayReason);
}

FTimerHandle UpdateHandle;
```

> Actor が Destroy された後もタイマーが残っているとコールバックで Dangling ポインタが発生しうる。`UObject` 弱参照ベースの内部実装によりある程度保護されるが、`EndPlay` でクリアするのが安全。

---

## タイムスケール連動

```cpp
// WorldSettings のタイムダイレーション
AWorldSettings* WS = GetWorld()->GetWorldSettings();
WS->TimeDilation = 0.5f;  // スローモーション（0.5x）
// → FTimerManager のすべてのタイマーが 0.5x の速度で進む
```

`bIgnorePause = true` を付けてもタイムダイレーションの影響は受ける。タイムダイレーションも無視するには `bIgnorePause` に加え `GetWorld()->GetRealTimeSeconds()` を使った手動実装が必要。

---

## 関連ドキュメント

- [[a_delegate_types]] — FTimerDelegate の元となる TDelegate/TMulticastDelegate
- [[b_dynamic_delegates]] — SetTimerByFunctionName の内部（UFUNCTION 名解決）
- [[Reference/ref_delegate_api]] — FTimerManager / FTimerHandle API 詳細
