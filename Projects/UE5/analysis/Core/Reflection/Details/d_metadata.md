# メタデータ（UMETA・メタデータスペシファイア）

- 上位: [[Reflection/01_overview]]
- 関連: [[a_uclass]] | [[b_fproperty]] | [[c_ufunction]]
- ソース: `CoreUObject/Public/UObject/MetaData.h`, `CoreUObject/Public/UObject/Class.h`

---

## 概要

UE5 のメタデータは、`UCLASS`/`UPROPERTY`/`UFUNCTION`/`USTRUCT` マクロの `Meta=(...)` に記述する **エディタ向けの付加情報**。ToolTip・編集条件・検証ルール・カテゴリ名など、エディタ UI の挙動をコード側から制御する。

**ランタイムには存在しない** — クック時にメタデータは除去される（`WITH_EDITORONLY_DATA` で囲まれている）。

---

## メタデータの構造

```mermaid
graph TD
    UField -->|HasMetaData / GetMetaData| MD[UMetaData]
    MD --> KV["TMap<FName, FString>"]
    UClass --> MD
    FProperty --> MD
    UFunction --> MD
```

`UMetaData` は `UPackage` が所有するオブジェクトで、`UField` / `FField` に `FName → FString` のキー・バリューマップを提供する。

---

## メタデータの記述方法

```cpp
UCLASS(meta=(ToolTip="プレイヤーの基本キャラクター",
             ShortTooltip="プレイヤーキャラ",
             DisplayName="Player Character"))
class AMyCharacter : public ACharacter { ... };

UPROPERTY(EditAnywhere, Category="Stats",
    meta=(
        ClampMin="0.0",
        ClampMax="1000.0",
        UIMin="0.0",
        UIMax="500.0",
        EditCondition="bIsAlive",
        EditConditionHides,
        ToolTip="現在の HP"
    ))
float Health;

UFUNCTION(BlueprintCallable,
    meta=(
        DisplayName="ダメージを与える",
        ToolTip="指定量のダメージをこのキャラクターに与える",
        DefaultToSelf="Target",
        WorldContext="WorldContextObject"
    ))
void TakeDamage(float Amount);
```

---

## よく使うメタデータスペシファイア

### UPROPERTY 向け

| キー | 型 | 説明 |
|------|----|------|
| `ClampMin` / `ClampMax` | float/int | 値のクランプ（内部値も制限） |
| `UIMin` / `UIMax` | float/int | スライダーの表示範囲（内部値は制限しない） |
| `EditCondition` | C++ 式 | 指定式が `true` のときのみ編集可 |
| `EditConditionHides` | — | `EditCondition` が false のとき UI から非表示 |
| `InlineEditConditionToggle` | — | チェックボックスを隣に表示 |
| `ToolTip` | string | ホバー時のツールチップ |
| `DisplayName` | string | UI 上の表示名 |
| `Category` | string | `Category="..."` と同じ（メタでも指定可） |
| `AllowedClasses` | クラス名 | オブジェクト参照プロパティで選択可能なクラスを制限 |
| `RequiredAssetDataTags` | タグ式 | アセット選択フィルタ |
| `NoResetToDefault` | — | 右クリック「Reset to Default」を無効化 |
| `ShowOnlyInnerProperties` | — | 構造体の展開表示 |
| `TitleProperty` | プロパティ名 | 配列要素のタイトルに使うプロパティ |
| `Units` | 単位文字列 | `"cm"` / `"kg"` など（数値の隣に表示） |
| `ForceUnits` | 単位文字列 | 強制的に単位変換 |
| `MakeStructureDefaultValue` | — | デフォルト値を構造体のコンストラクタから取得 |

### UFUNCTION 向け

| キー | 説明 |
|------|------|
| `DisplayName` | BP ノードの表示名 |
| `ToolTip` | BP ノードのツールチップ |
| `DefaultToSelf` | 指定引数のデフォルトを self に |
| `WorldContext` | ワールドコンテキスト引数名 |
| `ExpandEnumAsExecs` | 列挙型引数を実行ピンに展開 |
| `LatentInfo` | 非同期ノードの情報引数名 |
| `Latent` | Latent（遅延）ノードとしてマーク |
| `HidePin` | 指定引数ピンを隠す |
| `AutoCreateRefTerm` | 参照引数に自動デフォルトを生成 |

### UCLASS 向け

| キー | 説明 |
|------|------|
| `ToolTip` / `ShortTooltip` | クラスのツールチップ |
| `DisplayName` | エディタ上の表示名 |
| `ClassGroup` | アクタ追加ダイアログのグループ |
| `HideCategories` | 指定カテゴリを Details から非表示 |
| `ShowCategories` | 親クラスで非表示になったカテゴリを再表示 |
| `ComponentWrapperClass` | コンポーネントラッパーとしてマーク |

---

## EditCondition — 条件付き編集

```cpp
UPROPERTY(EditAnywhere)
bool bUseCustomSpeed;

UPROPERTY(EditAnywhere, meta=(EditCondition="bUseCustomSpeed", EditConditionHides))
float CustomSpeed;
// bUseCustomSpeed が false のとき CustomSpeed は非表示
```

`EditCondition` には `!bSomething` や `EnumValue == EMyEnum::Active` のような C++ 式が使える（評価はエディタが行う）。

---

## ランタイムでのメタデータアクセス

エディタビルドのみ。`WITH_EDITOR` or `WITH_EDITORONLY_DATA` で囲む必要がある:

```cpp
#if WITH_EDITOR
FProperty* Prop = UMyClass::StaticClass()->FindPropertyByName(TEXT("Health"));

// キーが存在するか
bool bHas = Prop->HasMetaData(TEXT("ClampMin"));

// 値を取得
FString ClampMin = Prop->GetMetaData(TEXT("ClampMin"));
float MinVal = FCString::Atof(*ClampMin);

// 設定（通常は UHT が行う。手動設定は稀）
Prop->SetMetaData(TEXT("ToolTip"), TEXT("カスタムツールチップ"));
#endif
```

---

## UMETA — Enum 向けメタデータ

```cpp
UENUM(BlueprintType)
enum class EMyState : uint8
{
    Idle    UMETA(DisplayName="待機中"),
    Moving  UMETA(DisplayName="移動中", ToolTip="移動状態"),
    Dead    UMETA(DisplayName="死亡", Hidden),  // BP のドロップダウンから非表示
};
```

---

## カスタムメタデータの追加

UHT の解析を通じてカスタムキーを追加できる（エディタ UI 上での解釈は自分で実装する必要あり）:

```cpp
UPROPERTY(meta=(MyCustomKey="CustomValue"))
int32 MyProp;

// 独自エディタ Customization で読む
FString Val = Prop->GetMetaData(TEXT("MyCustomKey"));
```

---

## 備考

- **ランタイム非存在**: クック後のビルドにはメタデータが含まれないため、ゲームコードで `GetMetaData` を呼ぶと `WITH_EDITOR` でのみコンパイル可
- **UHT がバリデーション**: 不正なメタデータキーや値はビルドエラーになることがある
- **Blueprint のピン名**: `DisplayName` メタはネイティブ関数の BP ノード表示名に使われ、日本語も使用可能

---

## 関連ドキュメント

- [[a_uclass]] — `UCLASS(meta=...)` の適用先
- [[b_fproperty]] — `UPROPERTY(meta=...)` の適用先
- [[c_ufunction]] — `UFUNCTION(meta=...)` の適用先
- [[Reference/ref_macros]] — UCLASS/UPROPERTY/UFUNCTION の全指定子
