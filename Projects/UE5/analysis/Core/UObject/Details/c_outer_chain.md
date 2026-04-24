# Outer チェーンとパス解決

- 上位: [[UObject/01_overview]]
- 関連: [[a_lifecycle]] | [[d_class_default_object]]
- ソース: `CoreUObject/Public/UObject/UObjectBaseUtility.h`, `CoreUObject/Public/UObject/UObjectGlobals.h`, `CoreUObject/Public/UObject/Object.h`

---

## 概要

すべての `UObject` は **Outer ポインタ**（所有者）を持ち、`UPackage` をルートとするツリー構造を形成する。このツリーにより、各オブジェクトは `/Game/Path/Package.Object:SubObject` 形式のフルパスで一意に識別され、アセット管理・GC・シリアライゼーションに活用される。

---

## Outer チェーン構造

```
UPackage: /Game/Blueprints/BP_Character
  └─ UBlueprint: BP_Character
       ├─ UBlueprintGeneratedClass: BP_Character_C
       │    └─ UFunction: MyFunction
       └─ UObject: AssetImportData
```

各オブジェクトは `OuterPrivate` に直接の親を保持し、チェーンをたどることで `UPackage` に到達できる。

---

## パス形式

| 形式 | 例 | 取得メソッド |
|------|-----|-------------|
| **パスのみ** | `/Game/Characters/BP_Character.BP_Character` | `GetPathName()` |
| **フルネーム** | `Blueprint /Game/Characters/BP_Character.BP_Character` | `GetFullName()` |
| **サブオブジェクト** | `/Game/Characters/BP_Character.BP_Character:MySubObj` | `GetPathName()` |
| **パッケージ名** | `/Game/Characters/BP_Character` | `GetOutermost()->GetName()` |

`:` で区切られた部分がサブオブジェクト（パッケージ内の他オブジェクトが所有する場合）。

---

## 主要 API

```cpp
// 直接の Outer を取得
UObject* Outer = MyObj->GetOuter();

// 最上位（UPackage）まで登る
UPackage* Package = MyObj->GetOutermost();

// Outer のうち特定型を探す
UBlueprintGeneratedClass* OwnerClass = MyObj->GetTypedOuter<UBlueprintGeneratedClass>();

// フルパス取得
FString Path = MyObj->GetPathName();     // /Game/Foo/Bar.Bar
FString Full = MyObj->GetFullName();     // Blueprint /Game/Foo/Bar.Bar

// パスからオブジェクトを探す（ロード済みのみ）
UObject* Found = StaticFindObject(UObject::StaticClass(), nullptr, TEXT("/Game/Foo/Bar.Bar"));

// パスからロード（未ロードなら非同期ロード可）
UObject* Loaded = StaticLoadObject(UObject::StaticClass(), nullptr, TEXT("/Game/Foo/Bar.Bar"));
```

---

## GetTransientPackage()

一時的な UObject（テスト・ランタイム生成で保存不要）は `GetTransientPackage()` を Outer にする:

```cpp
UMyObject* TempObj = NewObject<UMyObject>(GetTransientPackage(), TEXT("TempObj"),
    RF_Transient);
```

`GetTransientPackage()` は `/Engine/Transient` パッケージで、シリアライズされない。`RF_Transient` フラグと組み合わせることが多い。

---

## Rename（オブジェクトの移動・名前変更）

```cpp
// 名前を変更する
MyObj->Rename(TEXT("NewName"), NewOuter, REN_None);
// → Outer が変わると GetPathName() の結果も変わる
// → GUObjectHash が自動更新される

// REN フラグ
// REN_DoNotDirty  — パッケージを Dirty にしない
// REN_ForceNoResetLoaders — Linker をリセットしない
// REN_NonTransactional — Undo 対象にしない
```

`Rename()` は参照先のパスも更新する必要があるため、通常はエディタ操作（アセットリネーム等）経由でのみ使う。

---

## Outer と GC の関係

**Outer が生きていれば子孫も生きている** わけではない。GC はあくまで参照グラフを使う。ただし慣習的に Outer が所有者なのでセットで生存していることが多い。

- `UPackage` 自体が `AddToRoot()` されているか、何かから参照されていないと GC される
- `RF_Standalone` フラグを持つ UPackage は参照がなくても存在し続ける（ロード済みアセット）

---

## パッケージのパスとゲームコンテンツディレクトリ

| プレフィクス | 意味 |
|-------------|------|
| `/Game/...` | プロジェクトの `Content/` フォルダ |
| `/Engine/...` | エンジンの `Engine/Content/` フォルダ |
| `/Plugin/...` | プラグインの `Content/` フォルダ |
| `/Script/ModuleName` | C++ モジュール（UClass が属するパッケージ） |
| `/Engine/Transient` | 一時パッケージ |

---

## IsA / IsChildOf

型チェックには Outer チェーンではなくクラス継承チェーンを使う:

```cpp
bool bIsActor = Obj->IsA<AActor>();
bool bIsChild = MyClass->IsChildOf(AActor::StaticClass());
bool bImplements = Obj->GetClass()->ImplementsInterface(UMyInterface::StaticClass());
```

---

## 関連ドキュメント

- [[a_lifecycle]] — `NewObject` の Outer 指定
- [[Reference/ref_uobject_api]] — `GetPathName` / `GetOutermost` / `Rename` のAPI
