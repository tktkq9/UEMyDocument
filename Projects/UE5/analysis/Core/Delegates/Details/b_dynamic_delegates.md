# Dynamic デリゲート（Blueprint 連携）

- 上位: [[Delegates/01_overview]]
- 関連: [[a_delegate_types]] | [[c_timer]]
- ソース: `Core/Public/Delegates/DelegateBase.h`, `Core/Public/UObject/ScriptDelegates.h`

---

## 概要

`TDynamicDelegate` / `TDynamicMulticastDelegate` は **リフレクション経由**（`UFunction` 名ベース）でバインドする。Blueprint から割り当て可能な代わりに、通常デリゲートより実行コストが高い（文字列→関数ポインタの解決が発生）。

---

## DECLARE_DYNAMIC_DELEGATE マクロ群

```cpp
// Blueprint で設定・呼び出しできる単体デリゲート
DECLARE_DYNAMIC_DELEGATE(FMyDynDelegate)
DECLARE_DYNAMIC_DELEGATE_RetVal(bool, FMyDynBoolDelegate)
DECLARE_DYNAMIC_DELEGATE_OneParam(FMyDynOneParam, int32, ParamName)
DECLARE_DYNAMIC_DELEGATE_TwoParams(FMyDynTwoParams, int32, A, FString, B)

// Blueprint で設定できるマルチキャストデリゲート（BlueprintAssignable）
DECLARE_DYNAMIC_MULTICAST_DELEGATE(FMyDynMulticast)
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnHealthChanged, float, NewHealth)
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnDamage, float, DamageAmount, AActor*, Instigator)
```

> 通常デリゲートとの違い: 引数に **変数名** が必要（Blueprint エディタでの表示名として使われる）。

---

## Blueprint Assignable — イベントディスパッチャー

```cpp
UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    // BlueprintAssignable: BP で += できるイベント
    UPROPERTY(BlueprintAssignable, Category="Events")
    FOnHealthChanged OnHealthChanged;

    // BlueprintCallable: BP からも Broadcast 可
    UPROPERTY(BlueprintAssignable, BlueprintCallable, Category="Events")
    FOnDamage OnDamage;
};
```

Blueprint では `OnHealthChanged` をノードとして取り出し、`Assign` ノードで関数をバインドできる。

---

## C++ からのバインドと Broadcast

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnHealthChanged, float, NewHealth);

// バインド
OnHealthChanged.AddDynamic(this, &AMyActor::HandleHealthChanged);

// 実行（全バインドを呼び出し）
OnHealthChanged.Broadcast(NewHealthValue);

// 解除
OnHealthChanged.RemoveDynamic(this, &AMyActor::HandleHealthChanged);

// 関数宣言：UFUNCTION が必要（Dynamic はリフレクション経由のため）
UFUNCTION()
void HandleHealthChanged(float NewHealth);
```

> `AddDynamic` のバインド先は必ず `UFUNCTION()` を付ける。付いていないとコンパイルエラーになる。

---

## TDynamicDelegate — 単体 Dynamic デリゲート

```cpp
DECLARE_DYNAMIC_DELEGATE_RetVal_OneParam(bool, FMyConditionCheck, int32, Value);

// UPROPERTY として公開（BP で関数を設定できる）
UPROPERTY(EditAnywhere, BlueprintReadWrite)
FMyConditionCheck ConditionCheck;

// C++ からバインド
ConditionCheck.BindDynamic(this, &AMyActor::CheckValue);

// 実行
if (ConditionCheck.IsBound())
{
    bool bResult = ConditionCheck.Execute(42);
}

// 関数側の宣言
UFUNCTION()
bool CheckValue(int32 Value) { return Value > 0; }
```

---

## 通常デリゲート vs Dynamic デリゲート

| 比較項目 | `TMulticastDelegate` | `TDynamicMulticastDelegate` |
|----------|---------------------|-----------------------------|
| バインド方法 | 関数ポインタ | `UFunction` 名（文字列→関数解決） |
| Blueprint 対応 | 不可 | `BlueprintAssignable` で可 |
| 実行コスト | 低（直接呼び出し） | 高（`ProcessEvent` 経由） |
| バインド先の制約 | `UFUNCTION` 不要 | `UFUNCTION` 必須 |
| シリアライズ | 不可 | 可（`UObject` と一緒に保存可能） |
| `UPROPERTY` | 不可 | 必須（Blueprint アクセスのため） |

---

## FScriptDelegate / FMulticastScriptDelegate

Runtime の下位型。通常は直接使わないが、動的バインドの内部実装として知っておくと役立つ:

```cpp
// 低レベル手動バインド（型安全なマクロが使えない場合）
FScriptDelegate ScriptDelegate;
ScriptDelegate.BindUFunction(TargetObject, FName(TEXT("MyFunction")));

FMulticastScriptDelegate MulticastDelegate;
MulticastDelegate.Add(ScriptDelegate);
MulticastDelegate.ProcessMulticastDelegate<UObject>(nullptr);  // 引数なし呼び出し
```

---

## Blueprint イベントディスパッチャーとの対応

Blueprint で作成した「Event Dispatcher」は `TDynamicMulticastDelegate` に相当する。C++ から `UPROPERTY(BlueprintAssignable)` で公開したデリゲートは BP 側でイベントディスパッチャーとして扱われる。

```
C++ UPROPERTY(BlueprintAssignable)  ←→  Blueprint Event Dispatcher
C++ OnHealthChanged.Broadcast(...)  ←→  BP "Call OnHealthChanged" ノード
BP の += (Bind) ノード             ←→  AddDynamic
```

---

## よくある間違い

```cpp
// NG: UFUNCTION なしで AddDynamic — コンパイルエラー
void HandleHealthChanged(float HP) { ... }
OnHealthChanged.AddDynamic(this, &AMyActor::HandleHealthChanged);  // エラー

// OK: UFUNCTION を必ず付ける
UFUNCTION()
void HandleHealthChanged(float HP) { ... }

// NG: 非 UObject から AddDynamic — AddDynamic は UObject 専用
// 非 UObject なら通常の AddLambda を使うか、設計を変える
```

---

## コード実行フロー

### エントリポイント（AddDynamic 〜 Broadcast 〜 ProcessEvent）

```
(Dynamic Multicast バインド)
OnHealthChanged.AddDynamic(this, &AMyActor::HandleHealthChanged)   [DynamicDelegate.h]
  └─ __Internal_AddDynamic(InUserObject, InMethodPtr, FunctionName)
       └─ TMulticastScriptDelegate::Add(FScriptDelegate)
            ├─ FScriptDelegate::BindUFunction(Object, FName)        [ScriptDelegates.h]
            │    ├─ Object = InUserObject (TWeakObjectPtr で保持)   ← UObject 弱参照
            │    └─ FunctionName = FName("HandleHealthChanged")
            └─ InvocationList.Add(NewDelegate)                       ← TArray<FScriptDelegate>

(Broadcast 実行)
OnHealthChanged.Broadcast(NewHealth)                                [DynamicDelegate.h]
  └─ TMulticastScriptDelegate::ProcessMulticastDelegate<UObject>(Params)
       │                                                             [ScriptDelegates.h]
       └─ for (FScriptDelegate& Delegate : InvocationList):
            └─ Delegate.ProcessDelegate<UObject>(Params)
                 ├─ if (!Object.IsValid()) skip                      ← GC されていれば除去
                 └─ UFunction* Func = Object->FindFunction(FunctionName)
                      └─ Object->ProcessEvent(Func, Params)          ← リフレクション呼出

(C++ 関数実体に到達)
UObject::ProcessEvent(Func, Params)                                 [UObject.cpp]
  └─ Func->Invoke(Object, Stack, ReturnValue)
       └─ Func->FUNC_Native_*  (UHT 生成 thunk)
            └─ HandleHealthChanged(NewHealth)                        ← ユーザー関数

(BlueprintAssignable - BP からのバインド)
BP "Bind Event to OnHealthChanged" ノード                           [Blueprint VM]
  └─ KismetCompiler が FScriptDelegate を生成
       └─ AddDynamic と同じ経路で InvocationList に追加

(SetTimerByFunctionName 経由)
GetWorldTimerManager().SetTimerByFunctionName(this, FName("MyCallback"), 2.0f)
  └─ FTimerDelegate Delegate                                         [TimerManager.cpp]
       └─ Delegate.BindUFunction(Object, FName)                      ← FScriptDelegate と同じ
  └─ SetTimer(Handle, Delegate, 2.0f, false)
```

### フロー詳細

1. **AddDynamic 内部** — マクロは `__Internal_AddDynamic` を展開し、`FScriptDelegate::BindUFunction` で UObject と関数名 (`FName`) のペアを `TMulticastScriptDelegate::InvocationList` に追加する。UObject は `TWeakObjectPtr` で保持されるので GC で解除される。
2. **UFUNCTION 必須の理由** — Dynamic デリゲートは関数名 (`FName`) でリフレクション解決するため、対象関数が `UFunction` リフレクション情報を持っている必要がある（[[Reflection/Details/c_ufunction]]）。
3. **Broadcast 経路** — `ProcessMulticastDelegate` が `InvocationList` を走査し、各 `FScriptDelegate::ProcessDelegate` を呼ぶ。無効な UObject は自動でスキップされる。
4. **ProcessEvent 呼出** — `Object->FindFunction(FunctionName)` で `UFunction*` を取得し、`UObject::ProcessEvent` でパラメータを Stack に積んで Native thunk を呼ぶ。これが「リフレクション経由」のコスト源（[[Reflection/Details/c_ufunction]]）。
5. **ネイティブ vs Dynamic** — 通常デリゲート（[[a_delegate_types]]）は関数ポインタ直接呼出だが、Dynamic は `FName 検索 + ProcessEvent + Stack 積み` のため一桁遅い。代わりに BP 連携・シリアライズが可能。
6. **シリアライズ可能な理由** — `FScriptDelegate` は `Object*` と `FName` だけなので `UPROPERTY` でセーブデータに含められる。通常デリゲートは関数ポインタなので保存不可。
7. **BlueprintAssignable** — BP から `Assign` ノードでバインド可能にする `UPROPERTY` 指定子。BP コンパイラが `FScriptDelegate` を生成して同じ `InvocationList` に追加するため、C++ と BP のバインドが混在できる。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `TDynamicDelegate::__Internal_BindDynamic` | `DynamicDelegate.h` | Dynamic 単体バインド（マクロ展開） |
| `TDynamicMulticastDelegate::__Internal_AddDynamic` | `DynamicDelegate.h` | Dynamic マルチバインド |
| `FScriptDelegate::BindUFunction` | `ScriptDelegates.h` | UObject + FName でバインド |
| `FScriptDelegate::ProcessDelegate` | `ScriptDelegates.h` | リフレクション経由実行 |
| `TMulticastScriptDelegate::ProcessMulticastDelegate` | `ScriptDelegates.h` | InvocationList 走査 Broadcast |
| `UObject::FindFunction` | `UObject.cpp` | FName から UFunction* 解決 |
| `UObject::ProcessEvent` | `UObject.cpp` | UFunction を Stack 経由で呼出 |

---

## 関連ドキュメント

- [[a_delegate_types]] — 通常デリゲート（TDelegate/TMulticastDelegate）
- [[c_timer]] — FTimerManager（FTimerDelegate はDynamic対応）
- [[Reference/ref_delegate_api]] — Dynamic デリゲート API 詳細
