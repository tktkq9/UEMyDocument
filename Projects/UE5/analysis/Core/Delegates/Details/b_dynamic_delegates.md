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

## 関連ドキュメント

- [[a_delegate_types]] — 通常デリゲート（TDelegate/TMulticastDelegate）
- [[c_timer]] — FTimerManager（FTimerDelegate はDynamic対応）
- [[Reference/ref_delegate_api]] — Dynamic デリゲート API 詳細
