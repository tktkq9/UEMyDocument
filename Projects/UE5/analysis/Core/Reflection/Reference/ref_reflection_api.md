# Reflection API リファレンス（UClass / FProperty / UFunction）

- 上位: [[Reflection/01_overview]]
- 関連: [[Details/a_uclass]] | [[Details/b_fproperty]] | [[Details/c_ufunction]]
- ソース: `CoreUObject/Public/UObject/Class.h`, `CoreUObject/Public/UObject/Field.h`, `CoreUObject/Public/UObject/UnrealType.h`

---

## UClass API

### 取得

```cpp
// クラスオブジェクトを取得する方法
UClass* Class1 = UMyObject::StaticClass();      // 静的（型が分かっている場合）
UClass* Class2 = MyObj->GetClass();             // インスタンスから動的に取得
UClass* Class3 = FindObject<UClass>(ANY_PACKAGE, TEXT("MyObject"));  // 名前から

// クラス名
FString Name     = Class->GetName();            // "MyObject"
FString FriendlyName = Class->GetDescription(); // 説明文
```

### 継承チェック

```cpp
UClass* ParentClass = Class->GetSuperClass();
bool bIsChild  = Class->IsChildOf(UObject::StaticClass());
bool bIsChildOf = ChildClass->IsChildOf(ParentClass);

// 全祖先クラスをたどる
for (UClass* C = MyClass; C; C = C->GetSuperClass())
{
    UE_LOG(LogTemp, Log, TEXT("Parent: %s"), *C->GetName());
}
```

### CDO

```cpp
UObject* CDO = Class->GetDefaultObject();         // CDO 取得（生成がなければ作る）
UMyObject* TypedCDO = Class->GetDefaultObject<UMyObject>();
const UObject* CDOConst = Class->GetDefaultObject<UObject>(false);  // 生成しない
```

### インスタンス生成

```cpp
// クラスから動的にオブジェクト生成
UObject* Obj = NewObject<UObject>(Outer, TargetClass);
// または
UObject* Obj = StaticConstructObject_Internal(...);
```

### インターフェース確認

```cpp
bool bImplements = Class->ImplementsInterface(UMyInterface::StaticClass());

// インターフェースポインタ取得
IMyInterface* Iface = Cast<IMyInterface>(SomeObject);
```

---

## FProperty API

### プロパティのイテレート

```cpp
// クラスの全プロパティをイテレート（継承分含む）
for (TFieldIterator<FProperty> It(MyClass); It; ++It)
{
    FProperty* Prop = *It;
    FString Name    = Prop->GetName();
    uint64  Flags   = Prop->PropertyFlags;

    // SaveGame フラグ確認
    if (Prop->HasAnyPropertyFlags(CPF_SaveGame))
    {
        // ...
    }
}

// 特定のプロパティ型だけ取得
for (TFieldIterator<FFloatProperty> It(MyClass); It; ++It)
{
    FFloatProperty* FloatProp = *It;
}
```

### プロパティ検索

```cpp
FProperty* Prop = MyClass->FindPropertyByName(FName(TEXT("MyProp")));

// 型確認
if (FIntProperty* IntProp = CastField<FIntProperty>(Prop))
{
    // int32 型のプロパティ
}
if (FObjectProperty* ObjProp = CastField<FObjectProperty>(Prop))
{
    UClass* PropClass = ObjProp->PropertyClass;
}
```

### プロパティ値の読み書き

```cpp
// 生ポインタベースのアクセス（型を知っている場合）
FIntProperty* IntProp = CastField<FIntProperty>(Prop);
int32* ValuePtr = IntProp->ContainerPtrToValuePtr<int32>(TargetObject);
int32 CurrentVal = *ValuePtr;
*ValuePtr = 42;

// 型を知らない場合（FString 経由）
FString StrVal;
Prop->ExportTextItem_Direct(StrVal, Prop->ContainerPtrToValuePtr<void>(Obj), nullptr, nullptr, 0);

// ImportTextItem で文字列→値
const TCHAR* Remaining = nullptr;
Prop->ImportText_Direct(TEXT("42"), Prop->ContainerPtrToValuePtr<void>(Obj), nullptr, 0, GWarn);
```

### CopyCompleteValue

```cpp
// src から dst にプロパティ値をコピー
Prop->CopyCompleteValue(
    Prop->ContainerPtrToValuePtr<void>(DstObject),
    Prop->ContainerPtrToValuePtr<void>(SrcObject)
);

// デフォルト値と比較
bool bIsDefault = Prop->Identical_InContainer(Object, CDO);
```

### CPF フラグ

| フラグ | 説明 |
|--------|------|
| `CPF_Edit` | エディタで編集可 |
| `CPF_BlueprintVisible` | Blueprint から参照可 |
| `CPF_BlueprintReadOnly` | Blueprint から読み取り専用 |
| `CPF_Net` | ネットワーク複製対象 |
| `CPF_SaveGame` | SaveGame でシリアライズ |
| `CPF_Transient` | 保存しない |
| `CPF_NativeAccessSpecifierPublic` | C++ public |
| `CPF_OutParm` | 出力パラメータ |

---

## UFunction API

### 関数のイテレート・検索

```cpp
// クラスの全 UFunction をイテレート
for (TFieldIterator<UFunction> It(MyClass); It; ++It)
{
    UFunction* Func = *It;
    FString FuncName = Func->GetName();
    uint64  Flags    = Func->FunctionFlags;

    bool bCallable = Func->HasAnyFunctionFlags(FUNC_BlueprintCallable);
}

// 名前で検索
UFunction* Func = MyClass->FindFunctionByName(FName(TEXT("MyFunction")));
```

### ProcessEvent — 動的呼び出し

```cpp
// パラメータを準備（スタック構造体）
struct FMyFunctionParams
{
    int32 InValue;
    float ReturnValue;
};

FMyFunctionParams Params;
Params.InValue = 42;

UFunction* Func = MyObj->FindFunctionChecked(TEXT("MyFunction"));
MyObj->ProcessEvent(Func, &Params);

float Result = Params.ReturnValue;
```

### FUNC フラグ

| フラグ | 説明 |
|--------|------|
| `FUNC_Final` | オーバーライド不可 |
| `FUNC_Native` | C++ ネイティブ実装 |
| `FUNC_Event` | Blueprint イベント |
| `FUNC_Static` | 静的関数 |
| `FUNC_BlueprintCallable` | BP から呼び出し可 |
| `FUNC_BlueprintEvent` | BP でオーバーライド可 |
| `FUNC_Net` | RPC |
| `FUNC_NetServer` | サーバー RPC |
| `FUNC_NetClient` | クライアント RPC |
| `FUNC_NetMulticast` | マルチキャスト RPC |

---

## メタデータアクセス

```cpp
// クラスのメタデータ取得
bool bHasMeta = Class->HasMetaData(TEXT("Blueprintable"));
FString MetaVal = Class->GetMetaData(TEXT("ToolTip"));

// プロパティのメタデータ
FString ClampMin = Prop->GetMetaData(TEXT("ClampMin"));

// 実行時メタデータ有効化が必要（Editor build のみデフォルト有効）
// WITH_EDITORONLY_DATA ガードが必要な場合あり
```

---

## TSubclassOf とリフレクション

```cpp
// TSubclassOf<T> はエディタで UClass* をドロップダウン選択可能にする
UPROPERTY(EditAnywhere)
TSubclassOf<AActor> ActorClass;

// 実行時に UClass* として扱う
UClass* Class = ActorClass.Get();
if (Class)
{
    AActor* Spawned = GetWorld()->SpawnActor<AActor>(Class);
}

// TSubclassOf の継承チェック
TSubclassOf<APawn> PawnClass = AMyCharacter::StaticClass();  // 自動的に確認
```

---

## 関連ドキュメント

- [[Details/a_uclass]] — UClass 継承・EClassFlags の概念
- [[Details/b_fproperty]] — FProperty 階層・メモリレイアウト
- [[Details/c_ufunction]] — UFunction・ProcessEvent フロー
- [[ref_macros]] — UCLASS/UPROPERTY/UFUNCTION マクロ一覧
