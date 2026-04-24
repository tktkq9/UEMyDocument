# TDelegate / TMulticastDelegate / FTimerManager API リファレンス

- 上位: [[Delegates/01_overview]]
- 関連: [[Details/a_delegate_types]] | [[Details/b_dynamic_delegates]] | [[Details/c_timer]]
- ソース: `Core/Public/Delegates/Delegate.h`, `Engine/Public/TimerManager.h`

---

## デリゲート宣言マクロ

```cpp
// 単体デリゲート
DECLARE_DELEGATE(FName)
DECLARE_DELEGATE_RetVal(RetType, FName)
DECLARE_DELEGATE_OneParam(FName, Param1Type)
DECLARE_DELEGATE_TwoParams(FName, P1Type, P2Type)
DECLARE_DELEGATE_ThreeParams(FName, P1, P2, P3)
// ... (最大 DECLARE_DELEGATE_NineParams まで)

// マルチキャスト
DECLARE_MULTICAST_DELEGATE(FName)
DECLARE_MULTICAST_DELEGATE_OneParam(FName, P1Type)
// ... (最大 9 引数)

// Dynamic 単体
DECLARE_DYNAMIC_DELEGATE(FName)
DECLARE_DYNAMIC_DELEGATE_RetVal(RetType, FName)
DECLARE_DYNAMIC_DELEGATE_OneParam(FName, P1Type, P1Name)
// ... (引数に変数名が必要)

// Dynamic マルチキャスト
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FName)
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FName, P1Type, P1Name)
// ...
```

---

## TDelegate — バインド API

```cpp
// バインド
void BindUObject(UObject* InUserObject, RetType (UserClass::*Func)(Params...));
void BindSP(TSharedRef<UserClass> InUserObjectRef, RetType (UserClass::*Func)(Params...));
void BindRaw(UserClass* InUserObject, RetType (UserClass::*Func)(Params...));
void BindLambda(FunctorType&& InFunctor);
void BindStatic(RetType (*InFunc)(Params...));
void BindUFunction(UObject* InUserObject, const FName& InFunctionName);
void BindWeakLambda(UObject* InUserObject, FunctorType&& InFunctor);

// 実行
RetType Execute(Params...) const;
void ExecuteIfBound(Params...) const;  // 戻り値なしのみ

// 状態
bool IsBound() const;
bool IsBoundToObject(const void* InUserObject) const;
void Unbind();
UObject* GetUObject() const;

// 生成（static）
static TDelegate<...> CreateUObject(...);
static TDelegate<...> CreateSP(...);
static TDelegate<...> CreateRaw(...);
static TDelegate<...> CreateLambda(...);
static TDelegate<...> CreateStatic(...);
static TDelegate<...> CreateUFunction(...);
static TDelegate<...> CreateWeakLambda(...);
```

---

## TMulticastDelegate — バインド API

```cpp
// バインド（FDelegateHandle を返す）
FDelegateHandle AddUObject(UObject* InUserObject, void (UserClass::*Func)(Params...));
FDelegateHandle AddSP(TSharedRef<UserClass> InUserObjectRef, void (UserClass::*Func)(Params...));
FDelegateHandle AddRaw(UserClass* InUserObject, void (UserClass::*Func)(Params...));
FDelegateHandle AddLambda(FunctorType&& InFunctor);
FDelegateHandle AddStatic(void (*InFunc)(Params...));
FDelegateHandle AddUFunction(UObject* InUserObject, const FName& InFunctionName);
FDelegateHandle AddWeakLambda(UObject* InUserObject, FunctorType&& InFunctor);

// 実行
void Broadcast(Params...) const;

// 削除
bool Remove(FDelegateHandle Handle);          // ハンドル指定
bool RemoveAll(const void* InUserObject);     // 特定オブジェクトのバインドを全削除
void Clear();                                 // 全削除

// 状態
bool IsBound() const;
bool IsBoundToObject(const void* InUserObject) const;
int32 Num() const;                            // バインド数

// Dynamic 版（UFUNCTION 必須）
void AddDynamic(UObject*, void (UserClass::*Func)(Params...));
void RemoveDynamic(UObject*, void (UserClass::*Func)(Params...));
bool IsAlreadyBound(UObject*, void (UserClass::*Func)(Params...)) const;
```

---

## FDelegateHandle

```cpp
// ハンドルの有効性確認
bool FDelegateHandle::IsValid() const;

// リセット
void FDelegateHandle::Reset();

// 比較
bool operator==(const FDelegateHandle& Other) const;
bool operator!=(const FDelegateHandle& Other) const;
```

---

## FTimerManager API

```cpp
// アクセス
FTimerManager& World->GetTimerManager();
FTimerManager& GetWorldTimerManager();  // AActor 内ショートカット

// タイマー設定
void SetTimer(
    FTimerHandle& InOutHandle,
    TFunction<void()> Callback,
    float Rate,
    bool bLoop,
    float FirstDelay = -1.0f,   // -1 = Rate と同じ
    bool bIgnorePause = false
);

void SetTimer(
    FTimerHandle& InOutHandle,
    UObject* InObj,
    void (UserClass::*InTimerMethod)(),
    float InRate,
    bool InbLoop = false,
    float InFirstDelay = -1.f
);

void SetTimer(
    FTimerHandle& InOutHandle,
    const FTimerDelegate& InDelegate,
    float InRate,
    bool InbLoop = false,
    float InFirstDelay = -1.f
);

// 次フレーム実行
void SetTimerForNextTick(TFunction<void()> Callback);
void SetTimerForNextTick(UObject* InObj, void (UserClass::*InTimerMethod)());
void SetTimerForNextTick(const FTimerDelegate& InDelegate);

// 関数名指定版（Blueprint互換）
void SetTimerByFunctionName(UObject* Object, FString FunctionName, float Time, bool bLooping, float FirstDelay = -1.f);

// タイマー操作
void ClearTimer(FTimerHandle& InHandle);
void ClearAllTimersForObject(void const* Object);
void PauseTimer(FTimerHandle InHandle);
void UnPauseTimer(FTimerHandle InHandle);

// タイマー状態の照会
bool IsTimerActive(FTimerHandle InHandle) const;
bool IsTimerPaused(FTimerHandle InHandle) const;
bool IsTimerPending(FTimerHandle InHandle) const;
bool TimerExists(FTimerHandle InHandle) const;
float GetTimerRate(FTimerHandle InHandle) const;
float GetTimerRemaining(FTimerHandle InHandle) const;
float GetTimerElapsed(FTimerHandle InHandle) const;
```

---

## FTimerDelegate

```cpp
// デリゲート型
DECLARE_DELEGATE(FTimerDelegate);

// 生成例
FTimerDelegate D1 = FTimerDelegate::CreateUObject(this, &AMyActor::OnTimer);
FTimerDelegate D2 = FTimerDelegate::CreateLambda([]() { DoSomething(); });
FTimerDelegate D3 = FTimerDelegate::CreateSP(SharedObj, &FMyClass::OnTimer);
```

---

## FTimerHandle

```cpp
// タイマーの識別子（コピー可）
FTimerHandle Handle;

bool Handle.IsValid() const;
void Handle.Invalidate();

// 比較
bool operator==(const FTimerHandle& Other) const;
```

---

## Dynamic デリゲートのバインド差分

| 操作 | 通常 | Dynamic |
|------|------|---------|
| バインド | `AddUObject` / `BindUObject` | `AddDynamic` / `BindDynamic` |
| 解除 | `Remove(Handle)` | `RemoveDynamic` |
| バインド先 | 任意メソッド | `UFUNCTION` のみ |
| UPROPERTY | 使えない | 必須（BP アクセスのため） |
| 実行 | `Broadcast` / `Execute` | 同じ |

---

## 関連ドキュメント

- [[Details/a_delegate_types]] — バインド方法・FDelegateHandle の使い方
- [[Details/b_dynamic_delegates]] — Blueprint Assignable・AddDynamic パターン
- [[Details/c_timer]] — FTimerManager の使用例・ライフタイム管理
