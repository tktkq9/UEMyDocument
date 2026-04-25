# FProperty — プロパティリフレクション

- 上位: [[Reflection/01_overview]]
- 関連: [[a_uclass]] | [[c_ufunction]]
- ソース: `CoreUObject/Public/UObject/UnrealType.h`（`FProperty : public FField`、UnrealType.h:173）

---

## 概要

`FProperty` は **UObject のメンバ変数のリフレクション情報**。UE5 で `UProperty`（UObject 派生）から `FField`/`FProperty`（非 UObject）へ変更され、GC 負荷を削減した。型別のサブクラス（`FNumericProperty`・`FObjectProperty`・`FArrayProperty` 等）がプロパティの読み書き・コピー・ネット複製を担う。

---

## FProperty 階層

```mermaid
classDiagram
    class FField {
        +Owner: FFieldVariant
        +Next: FField*
        +NamePrivate: FName
        +FlagsPrivate: EObjectFlags
    }
    class FProperty {
        +ElementSize: int32
        +ArrayDim: int32
        +PropertyFlags: EPropertyFlags
        +RepIndex: uint16
        +CopyCompleteValue()
        +InitializeValue()
        +DestroyValue()
        +ExportTextItem()
        +ImportText()
    }
    class FNumericProperty
    class FInt8Property
    class FInt16Property
    class FIntProperty
    class FInt64Property
    class FFloatProperty
    class FDoubleProperty
    class FBoolProperty {
        +FieldMask: uint8
        +ByteMask: uint8
    }
    class FStrProperty
    class FNameProperty
    class FTextProperty
    class FObjectProperty {
        +PropertyClass: UClass*
    }
    class FWeakObjectProperty
    class FSoftObjectProperty
    class FClassProperty {
        +MetaClass: UClass*
    }
    class FStructProperty {
        +Struct: UScriptStruct*
    }
    class FArrayProperty {
        +Inner: FProperty*
    }
    class FMapProperty {
        +KeyProp: FProperty*
        +ValueProp: FProperty*
    }
    class FSetProperty {
        +ElementProp: FProperty*
    }
    class FEnumProperty {
        +Enum: UEnum*
        +UnderlyingProp: FNumericProperty*
    }
    class FDelegateProperty
    class FMulticastDelegateProperty

    FField <|-- FProperty
    FProperty <|-- FNumericProperty
    FNumericProperty <|-- FInt8Property
    FNumericProperty <|-- FIntProperty
    FNumericProperty <|-- FFloatProperty
    FNumericProperty <|-- FDoubleProperty
    FProperty <|-- FBoolProperty
    FProperty <|-- FStrProperty
    FProperty <|-- FNameProperty
    FProperty <|-- FTextProperty
    FProperty <|-- FObjectProperty
    FObjectProperty <|-- FWeakObjectProperty
    FObjectProperty <|-- FSoftObjectProperty
    FObjectProperty <|-- FClassProperty
    FProperty <|-- FStructProperty
    FProperty <|-- FArrayProperty
    FProperty <|-- FMapProperty
    FProperty <|-- FSetProperty
    FProperty <|-- FEnumProperty
    FProperty <|-- FDelegateProperty
    FProperty <|-- FMulticastDelegateProperty
```

---

## EPropertyFlags（CPF_*）主要フラグ

| フラグ | 値 | UPROPERTY 指定子 | 意味 |
|--------|-----|----------------|------|
| `CPF_Edit` | `0x0001` | `EditAnywhere`/`EditDefaultsOnly` | エディタ編集可 |
| `CPF_BlueprintVisible` | `0x0004` | `BlueprintReadWrite`/`BlueprintReadOnly` | Blueprint 公開 |
| `CPF_BlueprintReadOnly` | `0x0010` | `BlueprintReadOnly` | Blueprint 読み取り専用 |
| `CPF_Net` | `0x0020` | `Replicated` | ネット複製対象 |
| `CPF_Transient` | `0x2000` | `Transient` | シリアライズしない |
| `CPF_Config` | `0x4000` | `Config` | .ini から読み込む |
| `CPF_EditConst` | `0x20000` | `VisibleAnywhere` | 表示のみ（編集不可） |
| `CPF_SaveGame` | — | `SaveGame` | SaveGame シリアライズ対象 |
| `CPF_EditorOnly` | `0x800000000` | `meta=(...)` | エディタのみ |
| `CPF_DisableEditOnInstance` | `0x10000` | `EditDefaultsOnly` | インスタンス編集不可 |

---

## プロパティの操作

### TFieldIterator — クラスのプロパティ走査

```cpp
// UMyClass のすべての UPROPERTY を走査
for (TFieldIterator<FProperty> It(UMyClass::StaticClass()); It; ++It)
{
    FProperty* Prop = *It;
    UE_LOG(LogTemp, Log, TEXT("Property: %s, Type: %s"),
        *Prop->GetName(), *Prop->GetCPPType());
}

// 継承プロパティを含めるかどうか
TFieldIterator<FProperty> It(Class, EFieldIteratorFlags::IncludeSuper);
```

### 値の汎用取得・設定

```cpp
// インスタンスからプロパティ値を取得
FProperty* Prop = UMyClass::StaticClass()->FindPropertyByName(TEXT("Health"));
FFloatProperty* FloatProp = CastField<FFloatProperty>(Prop);
if (FloatProp)
{
    float Value = FloatProp->GetPropertyValue_InContainer(MyObj);
    FloatProp->SetPropertyValue_InContainer(MyObj, 200.f);
}
```

### CopyCompleteValue — 汎用コピー

```cpp
// src から dest へプロパティ値を型に関係なくコピー
Prop->CopyCompleteValue(DestContainer, SrcContainer);

// 初期値で埋める（コンストラクタ相当）
Prop->InitializeValue_InContainer(Container);

// デストラクタ相当
Prop->DestroyValue_InContainer(Container);
```

### テキスト入出力

```cpp
// FProperty の値をテキストにエクスポート
FString TextVal;
Prop->ExportTextItem_Direct(TextVal, PropPtr, nullptr, nullptr, PPF_None);

// テキストから値をインポート
const TCHAR* Text = TEXT("150.0");
Prop->ImportText_Direct(Text, PropPtr, nullptr, PPF_None);
```

---

## UE4 → UE5 の移行（UProperty → FProperty）

| 変更点 | UE4 | UE5 |
|--------|-----|-----|
| 基底 | `UProperty : public UField : public UObject` | `FProperty : public FField`（非 UObject） |
| GC | GC が追跡（コスト大） | GC 対象外（コスト削減） |
| キャスト | `Cast<UIntProperty>(Prop)` | `CastField<FIntProperty>(Prop)` |
| 型名 | `UIntProperty`/`UFloatProperty` | `FIntProperty`/`FFloatProperty` |
| 走査 | `TFieldIterator<UProperty>` | `TFieldIterator<FProperty>` |

---

## 主要サブクラスの特徴

### FBoolProperty

`bool` 型はビットフィールドのパッキングに対応:

```cpp
FBoolProperty* BoolProp = CastField<FBoolProperty>(Prop);
bool Val = BoolProp->GetPropertyValue_InContainer(Obj);
// 内部的には FieldMask と ByteMask でバイト内のビット位置を管理
```

### FObjectProperty

```cpp
FObjectProperty* ObjProp = CastField<FObjectProperty>(Prop);
UClass* PropClass = ObjProp->PropertyClass;  // どのクラスの UObject か
UObject* RefObj = ObjProp->GetObjectPropertyValue_InContainer(Obj);
```

`FWeakObjectProperty`・`FSoftObjectProperty`・`FLazyObjectProperty` は同様に特殊参照型の処理を持つ。

### FStructProperty

```cpp
FStructProperty* StructProp = CastField<FStructProperty>(Prop);
UScriptStruct* InnerStruct = StructProp->Struct;  // USTRUCT 型情報
void* StructData = StructProp->ContainerPtrToValuePtr<void>(Container);
```

---

## プロパティのネットワーク複製

`CPF_Net` フラグが立っている `FProperty` はレプリケーション対象。各プロパティは `RepIndex` を持ち、ネット複製テーブルで管理される。

```cpp
// RepIndex で RepLayout のプロパティを特定
uint16 RepIdx = Prop->RepIndex;
```

---

## コード実行フロー

### エントリポイント（FProperty 登録 〜 値アクセス 〜 シリアライズ）

```
(UHT 生成 〜 登録)
Z_Construct_UClass_UMyComponent()                                  [*.generated.cpp]
  └─ UECodeGen_Private::ConstructUClass()
       └─ NewProp_Health = new FFloatProperty(Class, "Health", ...) ← FProperty サブクラス生成
            └─ Class->AddCppProperty(NewProp)                        [Class.cpp]
                 └─ Children リンクリストに挿入
                      ├─ Property->Next = Children                   ← 単方向リスト
                      └─ Children = Property

(プロパティ検索)
UClass::FindPropertyByName(Name)                                   [Class.cpp]
  └─ for (FField* Field = Children; Field; Field = Field->Next)
       └─ if (Field->GetFName() == Name && Field is FProperty)
            └─ return (FProperty*)Field

(値の取得・設定)
FProperty::GetPropertyValue_InContainer(Container)                 [UnrealType.h]
  └─ ContainerPtrToValuePtr<T>(Container)
       └─ (uint8*)Container + Property->GetOffset_ForInternal()      ← オフセット計算
            └─ *(T*)Ptr 参照

(コピー)
FProperty::CopyCompleteValue(Dest, Src)                            [UnrealType.h]
  └─ for (Index = 0; Index < ArrayDim; ++Index)
       └─ CopySingleValue(DestPtr + ElementSize*Index, SrcPtr + ...) ← 型別実装

(シリアライズ)
UObject::Serialize(FArchive& Ar)                                   [Object.cpp]
  └─ Class->SerializeBin(Ar, this)                                  [Class.cpp]
       └─ for (FProperty* Prop = ...; Prop; Prop = Prop->Next)
            └─ Prop->SerializeBinProperty(Slot, Container)
                 └─ Prop->SerializeItem(Slot, ValuePtr)              ← 型別バイナリ化

(ネット複製)
FRepLayout::InitFromClass()                                        [RepLayout.cpp]
  └─ for each FProperty with CPF_Net flag:
       └─ RepLayoutCmds に追加（RepIndex でアクセス）
            └─ Channel->ReplicateActor() で差分送信
```

### フロー詳細

1. **FProperty 生成** — UHT 生成の `Z_Construct_UClass_*` が `new FIntProperty/FFloatProperty/...` でサブクラスを生成し、`AddCppProperty` で `UStruct::Children` リンクリストに挿入する。
2. **オフセット計算** — `FProperty::Offset_Internal` は C++ メンバの実バイトオフセット。`ContainerPtrToValuePtr<T>` がポインタ算術でアドレスを算出する。
3. **値の操作** — `GetPropertyValue_InContainer` / `SetPropertyValue_InContainer` が型安全に値を取り出す。`CastField<FFloatProperty>(Prop)` で型を確定する必要がある。
4. **CopyCompleteValue** — `ArrayDim` ループで `CopySingleValue` を呼ぶ。CDO → Instance のコピーで使われる（[[UObject/Details/d_class_default_object]]）。
5. **シリアライズ** — `UObject::Serialize` が `Class->SerializeBin` を呼び、各 `FProperty::SerializeBinProperty` が型別実装でバイナリ I/O を行う（[[Serialization/Details/a_farchive]]）。
6. **ネット複製** — `CPF_Net` フラグが立った `FProperty` は `FRepLayout` のコマンドリストに登録され、`RepIndex` でアクセスされる。
7. **FBoolProperty 特殊化** — ビットフィールド対応で `FieldMask`/`ByteMask` を持ち、バイト内のビット位置を管理する。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `UECodeGen_Private::ConstructUClass` | `UObjectGlobals.cpp` | FProperty 一括登録 |
| `UStruct::AddCppProperty` | `Class.cpp` | Children リスト挿入 |
| `UStruct::FindPropertyByName` | `Class.cpp` | 名前検索（O(N)） |
| `FProperty::ContainerPtrToValuePtr` | `UnrealType.h` | オフセット計算 |
| `FProperty::GetPropertyValue_InContainer` | `UnrealType.h` | 型安全アクセサ |
| `FProperty::CopyCompleteValue` / `CopySingleValue` | `UnrealType.h` | プロパティコピー |
| `FProperty::SerializeBinProperty` | `PropertyBinaryArchive.cpp` | バイナリ I/O |
| `FRepLayout::InitFromClass` | `RepLayout.cpp` | ネット複製レイアウト構築 |

---

## 関連ドキュメント

- [[a_uclass]] — プロパティを所有する `UClass`/`UStruct`
- [[c_ufunction]] — 引数・戻り値も `FProperty` で表現
- [[Reference/ref_reflection_api]] — `FProperty` / `TFieldIterator` の API
