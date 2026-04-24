# デリゲート型（TDelegate / TMulticastDelegate）

- 上位: [[Delegates/01_overview]]
- 関連: [[b_dynamic_delegates]] | [[c_timer]]
- ソース: `Core/Public/Delegates/Delegate.h`, `Core/Public/Delegates/MulticastDelegate.h`

---

## デリゲート型の選択

| 型 | バインド数 | 戻り値 | Blueprint | 用途 |
|----|-----------|--------|-----------|------|
| `TDelegate<R(Args...)>` | 1 | あり | 不可 | コールバック 1:1 |
| `TMulticastDelegate<void(Args...)>` | N | なし | 不可 | イベント通知 1:N |
| `TDynamicDelegate` | 1 | あり | 可 | BP 連携 1:1 |
| `TDynamicMulticastDelegate` | N | なし | 可 | BP 連携 1:N |
| `TFunction<R(Args...)>` | ラムダ専用 | あり | 不可 | 一時コールバック |

---

## DECLARE_DELEGATE マクロ群

```cpp
// 単体デリゲートの宣言
DECLARE_DELEGATE(FMyDelegate)                           // 引数なし・戻り値なし
DECLARE_DELEGATE_RetVal(bool, FMyBoolDelegate)          // 戻り値あり
DECLARE_DELEGATE_OneParam(FMyOneParam, int32)           // 引数1個
DECLARE_DELEGATE_TwoParams(FMyTwoParams, int32, FString)
// ...最大 9 引数

// マルチキャストの宣言
DECLARE_MULTICAST_DELEGATE(FMyMulticastDelegate)
DECLARE_MULTICAST_DELEGATE_OneParam(FMyEventOneParam, float)
DECLARE_MULTICAST_DELEGATE_TwoParams(FMyEvent, int32, bool)
```

マクロは `typedef TDelegate<...>` / `typedef TMulticastDelegate<...>` の別名を生成する。

---

## TDelegate — 単体デリゲート

### バインド方法

```cpp
DECLARE_DELEGATE_RetVal_OneParam(int32, FMyDelegate, float);

FMyDelegate MyDelegate;

// UObject メソッドバインド（弱参照。UObject が GC されたら自動解除）
MyDelegate.BindUObject(this, &AMyActor::OnEvent);

// UObject メソッド（SharedRef 版）
MyDelegate.BindSP(SharedObj.ToSharedRef(), &FMyClass::OnEvent);

// 生ポインタ（ライフタイム管理は手動）
MyDelegate.BindRaw(RawPtr, &FMyClass::OnEvent);

// ラムダ
MyDelegate.BindLambda([](float Val) -> int32 { return (int32)Val; });

// 静的関数
MyDelegate.BindStatic(&FreeFunction);
```

### 実行

```cpp
if (MyDelegate.IsBound())
{
    int32 Result = MyDelegate.Execute(3.14f);
}

// 安全な実行（バインドされていなければ何もしない・戻り値なしデリゲート専用）
MyDelegate.ExecuteIfBound();

// バインド確認
bool bBound = MyDelegate.IsBound();

// 解除
MyDelegate.Unbind();
```

---

## TMulticastDelegate — マルチキャストデリゲート

### バインド / 実行

```cpp
DECLARE_MULTICAST_DELEGATE_OneParam(FOnHealthChanged, float);

FOnHealthChanged OnHealthChanged;

// バインド（戻り値は FDelegateHandle — 解除に使う）
FDelegateHandle Handle = OnHealthChanged.AddUObject(this, &AMyActor::HandleHealthChange);
OnHealthChanged.AddLambda([](float HP) { UE_LOG(LogGame, Log, TEXT("HP: %f"), HP); });

// 全バインドを一斉に呼び出す（Broadcast）
OnHealthChanged.Broadcast(NewHealthValue);

// バインド解除（ハンドルで指定）
OnHealthChanged.Remove(Handle);

// 全解除
OnHealthChanged.Clear();

// バインド有無
bool bBound = OnHealthChanged.IsBound();
```

### FDelegateHandle

```cpp
// クラスにハンドルを保持して後で解除
class AMyActor : public AActor
{
    FDelegateHandle MyHandle;

    virtual void BeginPlay() override
    {
        MyHandle = SomeSubsystem->OnEvent.AddUObject(this, &AMyActor::HandleEvent);
    }

    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override
    {
        SomeSubsystem->OnEvent.Remove(MyHandle);
    }
};
```

---

## バインド関数の比較

| 関数 | 対象 | ライフタイム管理 |
|------|------|----------------|
| `BindUObject` / `AddUObject` | `UObject` 派生 | 弱参照（GC で自動解除） |
| `BindSP` / `AddSP` | `TSharedPtr` 管理オブジェクト | 弱参照（TWeakPtr 経由） |
| `BindRaw` / `AddRaw` | 生ポインタ | 手動管理（Dangling 危険） |
| `BindLambda` / `AddLambda` | ラムダ/std::function | コピーキャプチャに注意 |
| `BindStatic` / `AddStatic` | 静的/グローバル関数 | ライフタイムなし |
| `BindUFunction` / `AddUFunction` | UFunction（リフレクション経由） | UObject 弱参照 |

---

## TFunction — 軽量一時コールバック

`TDelegate` より軽量。ラムダやコールバックを変数に持ちたい場合:

```cpp
#include "Misc/TVariant.h"

TFunction<void(int32)> Callback = [](int32 Val)
{
    UE_LOG(LogTemp, Log, TEXT("Value: %d"), Val);
};

Callback(42);

// 空チェック
if (Callback)
{
    Callback(0);
}

// 引数付き関数の格納
TFunction<int32(float, float)> MathFunc = [](float A, float B) -> int32
{
    return (int32)(A + B);
};
```

`std::function<T>` と互換。UE コードでは `TFunction` を優先する。

---

## デリゲートとスレッド

デリゲートの `Broadcast` / `Execute` は **呼び出したスレッド上で** バインドを実行する。バインド先が GameThread 専用処理の場合は注意:

```cpp
// バックグラウンドスレッドからの Broadcast は避ける
// GameThread が必要なら AsyncTask で包む
AsyncTask(ENamedThreads::GameThread, [this]()
{
    OnDataReady.Broadcast(Data);
});
```

---

## 関連ドキュメント

- [[b_dynamic_delegates]] — Blueprint 対応 Dynamic デリゲート
- [[c_timer]] — FTimerManager・SetTimer（デリゲート経由のタイマー）
- [[Reference/ref_delegate_api]] — TDelegate / TMulticastDelegate API 詳細
