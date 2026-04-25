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

## コード実行フロー

### エントリポイント（パス解決・Rename・検索）

```
(パス取得)
UObject::GetPathName(StopOuter)                                   [UObjectBaseUtility.cpp]
  └─ GetPathName(StopOuter, StringBuilder)
       ├─ if (Outer != StopOuter && Outer != nullptr):
       │    └─ Outer->GetPathName(StopOuter, Out)                  ← 再帰で Outer チェーンを上る
       │    └─ Out.Append(SubObjectDelimiter ":" or ".")
       └─ Out.Append(GetName())                                     ← FName を末尾に追加

(パス → オブジェクト解決)
StaticFindObject(Class, Outer, Name, bExactClass)                 [UObjectGlobals.cpp]
  └─ StaticFindObjectFast()
       └─ FUObjectHashTables::Get().HashObjects.Find(Hash)         ← Outer+Name のハッシュ
            └─ 見つかれば UObject* を返す（ロード済みのみ）

StaticLoadObject(Class, Outer, Name)                              [UObjectGlobals.cpp]
  └─ StaticFindObject() で先に検索
       └─ 未ヒットなら StaticLoadObjectInternal()
            └─ LoadPackageInternal()                               ← 必要なら同期ロード
                 └─ FLinkerLoad で .uasset から取得

(Rename - 移動)
UObject::Rename(NewName, NewOuter, Flags)                         [Object.cpp]
  ├─ FUObjectHashTables::Get().RemoveObject(this)                  ← 旧 Hash 解除
  ├─ ClassPrivate->Rename(...)                                     ← サブオブジェクトも再帰
  ├─ NamePrivate = NewName / OuterPrivate = NewOuter
  └─ FUObjectHashTables::Get().AddObject(this)                     ← 新 Hash 登録

(Outer 検索)
UObject::GetTypedOuter<T>()                                       [UObjectBaseUtility.h]
  └─ for (Outer = GetOuter(); Outer; Outer = Outer->GetOuter())
       └─ if (Outer->IsA<T>()) return (T*)Outer                    ← 型一致まで上る
```

### フロー詳細

1. **パス組み立て** — `GetPathName` は Outer チェーンを再帰的に上って文字列を組み立てる。サブオブジェクト境界（パッケージ内オブジェクトの所有関係）は `:` で区切る。
2. **ハッシュ検索** — `StaticFindObject` は `FUObjectHashTables` の `HashObjects` で Outer+Name のハッシュ一致を見る。`O(1)` でロード済みオブジェクトを取得。
3. **ロード経路** — `StaticLoadObject` は未ロードなら `LoadPackageInternal` を経て `FLinkerLoad` を起動し、パッケージから対象 UObject を復元（[[Serialization/Details/b_asset_serialization]]）。
4. **Rename** — `Rename` は旧 Hash を `FUObjectHashTables` から外し、Outer/Name を更新したうえで新 Hash を登録する。サブオブジェクトもツリー全体で再ハッシュ。
5. **TypedOuter 検索** — `GetTypedOuter<T>()` は Outer をループでたどり、最初に `IsA<T>()` を満たす祖先を返す。`UPackage` まで一致しなければ `nullptr`。
6. **GetTransientPackage** — `/Engine/Transient` を返す。`RF_Transient` 付きで `NewObject` する一時 UObject の Outer に使う（[[a_lifecycle]]）。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `UObjectBaseUtility::GetPathName` / `GetFullName` | `UObjectBaseUtility.cpp` | フルパス組み立て |
| `UObjectBaseUtility::GetTypedOuter` | `UObjectBaseUtility.h` | 型指定 Outer 検索 |
| `StaticFindObject` / `StaticFindObjectFast` | `UObjectGlobals.cpp` | パス → オブジェクト解決 |
| `StaticLoadObject` / `LoadPackageInternal` | `UObjectGlobals.cpp` | 必要時にロード |
| `UObject::Rename` | `Object.cpp` | Outer/Name の付け替え |
| `FUObjectHashTables::AddObject` / `RemoveObject` | `UObjectHash.cpp` | 名前ハッシュ管理 |
| `GetTransientPackage` | `UObjectGlobals.cpp` | 一時パッケージ取得 |

---

## 関連ドキュメント

- [[a_lifecycle]] — `NewObject` の Outer 指定
- [[Reference/ref_uobject_api]] — `GetPathName` / `GetOutermost` / `Rename` のAPI
