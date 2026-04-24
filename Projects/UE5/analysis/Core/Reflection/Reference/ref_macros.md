# UCLASS / UPROPERTY / UFUNCTION / USTRUCT マクロリファレンス

- 上位: [[Reflection/01_overview]]
- 関連: [[Details/a_uclass]] | [[Details/b_fproperty]] | [[Details/c_ufunction]] | [[Details/d_metadata]]
- ソース: `CoreUObject/Public/UObject/ObjectMacros.h`

---

## UCLASS スペシファイア一覧

```cpp
UCLASS([specifier, ...])
class MYMODULE_API UMyClass : public UObject { GENERATED_BODY() };
```

| スペシファイア | 説明 |
|--------------|------|
| `Blueprintable` | Blueprint で継承可 |
| `NotBlueprintable` | Blueprint 継承を禁止 |
| `BlueprintType` | Blueprint で変数の型として使用可 |
| `Abstract` | 直接インスタンス化不可 |
| `EditInlineNew` | 別のオブジェクトのプロパティとして inline で作成可 |
| `Placeable` | レベルエディタにドラッグ&ドロップ可（Actor 向け） |
| `NotPlaceable` | エディタで配置不可 |
| `DefaultToInstanced` | UPROPERTY 使用時デフォルトでインスタンス化 |
| `Transient` | このクラスのオブジェクトは保存しない |
| `Config(IniName)` | Config ファイルからデフォルト値を読む |
| `Within=OuterClass` | 指定 Outer の中にのみ存在可能 |
| `ClassGroup(GroupName)` | エディタのグループ分類 |
| `HideDropdown` | Blueprint でドロップダウン選択から隠す |
| `CollapseCategories` | プロパティカテゴリを折りたたむ |
| `DontAutoCollapseCategories(Cat)` | 折りたたみ例外カテゴリ指定 |
| `MinimalAPI` | 最低限の API エクスポートのみ |
| `Deprecated` | 非推奨クラス |
| `ConversionRoot` | Blueprint 変換ルート |

---

## UPROPERTY スペシファイア一覧

```cpp
UPROPERTY([specifier, ...])
int32 MyProp;
```

### 可視性・編集性

| スペシファイア | 説明 |
|--------------|------|
| `VisibleAnywhere` | 全コンテキストで表示（編集不可） |
| `VisibleDefaultsOnly` | CDO/Archetype でのみ表示 |
| `VisibleInstanceOnly` | インスタンスでのみ表示 |
| `EditAnywhere` | 全コンテキストで編集可 |
| `EditDefaultsOnly` | CDO/Archetype でのみ編集可 |
| `EditInstanceOnly` | インスタンスでのみ編集可 |
| `EditFixedSize` | 配列サイズを変更不可 |

### Blueprint アクセス

| スペシファイア | 説明 |
|--------------|------|
| `BlueprintReadWrite` | BP から読み書き可 |
| `BlueprintReadOnly` | BP から読み取り専用 |
| `BlueprintGetter=FuncName` | カスタム getter 関数 |
| `BlueprintSetter=FuncName` | カスタム setter 関数 |

### シリアライズ・GC

| スペシファイア | 説明 |
|--------------|------|
| `SaveGame` | SaveGame でシリアライズ対象（CPF_SaveGame） |
| `Transient` | 保存しない・GC で追跡 |
| `DuplicateTransient` | Duplicate 時にコピーしない |
| `TextExportTransient` | テキストエクスポート時に除外 |
| `NonTransactional` | Undo/Redo 対象外 |
| `SkipSerialization` | シリアライズをスキップ |

### ネットワーク

| スペシファイア | 説明 |
|--------------|------|
| `Replicated` | ネットワーク複製有効 |
| `ReplicatedUsing=FuncName` | 複製時のコールバック指定 |
| `NotReplicated` | 明示的に複製しない |
| `RepRetry` | 複製の再試行有効 |

### その他

| スペシファイア | 説明 |
|--------------|------|
| `Category="Cat|SubCat"` | エディタのカテゴリ |
| `AdvancedDisplay` | 詳細パネルで折りたたむ |
| `Instanced` | UObject プロパティをインスタンス化 |
| `Export` | エクスポートに含める |
| `Config` | Config ファイルに保存 |
| `GlobalConfig` | グローバル Config に保存 |
| `Interp` | Sequencer のアニメーション対象 |
| `AssetRegistrySearchable` | Asset Registry に情報をインデックス |
| `NoClear` | null 設定を禁止 |
| `AllowAbstract` | Abstract クラスも選択可 |
| `MustImplement=InterfaceName` | 実装が必要なインターフェース |

---

## UFUNCTION スペシファイア一覧

```cpp
UFUNCTION([specifier, ...])
void MyFunction();
```

| スペシファイア | 説明 |
|--------------|------|
| `BlueprintCallable` | Blueprint から呼び出し可 |
| `BlueprintPure` | 副作用なし・戻り値あり（exec ピンなし） |
| `BlueprintImplementableEvent` | C++ 宣言・BP でオーバーライド |
| `BlueprintNativeEvent` | C++ デフォルト実装あり・BP でオーバーライド可 |
| `CallInEditor` | エディタボタン（Actor 選択時） |
| `Category="Cat"` | エディタのカテゴリ |
| `Exec` | コンソールコマンドとして実行可 |
| `Server` | サーバー RPC |
| `Client` | クライアント RPC |
| `NetMulticast` | マルチキャスト RPC |
| `Reliable` | 信頼性のある RPC |
| `Unreliable` | 非信頼性 RPC |
| `WithValidation` | RPC 検証関数 (`_Validate`) あり |
| `SealedEvent` | Blueprint でのオーバーライドを禁止 |
| `ServiceRequest` | サービスリクエスト |
| `ServiceResponse` | サービスレスポンス |

---

## USTRUCT スペシファイア一覧

```cpp
USTRUCT([specifier, ...])
struct FMyStruct { GENERATED_BODY() };
```

| スペシファイア | 説明 |
|--------------|------|
| `BlueprintType` | Blueprint で変数型として使用可 |
| `Atomic` | アトミックにシリアライズ（全フィールド or 何もなし） |
| `NoExport` | UHT がヘッダを生成しない |
| `Immutable` | 変更不可構造体（ネイティブ型のラッパー向け） |

---

## UENUM スペシファイア

```cpp
UENUM(BlueprintType)
enum class EMyEnum : uint8
{
    Value1 UMETA(DisplayName="Value 1"),
    Value2 UMETA(ToolTip="Second value"),
    Value3 UMETA(Hidden),  // エディタで非表示
};
```

| UMETA スペシファイア | 説明 |
|--------------------|------|
| `DisplayName="..."` | エディタ表示名 |
| `ToolTip="..."` | ツールチップ |
| `Hidden` | エディタのドロップダウンから隠す |

---

## UINTERFACE

```cpp
// インターフェース宣言（2 クラス方式）
UINTERFACE(MinimalAPI, Blueprintable)
class UMyInterface : public UInterface { GENERATED_BODY() };

class IMyInterface
{
    GENERATED_BODY()
public:
    // 純粋仮想（C++ 側で必須実装）
    virtual void Execute() = 0;

    // BP でオーバーライド可（デフォルト実装あり）
    UFUNCTION(BlueprintNativeEvent)
    void OnEvent();
};
```

---

## UPARAM — 引数修飾子

```cpp
// Blueprint での引数名を制御
UFUNCTION(BlueprintCallable)
void MyFunc(UPARAM(ref) FMyStruct& InStruct,  // BP で参照渡し
            UPARAM(DisplayName="Count") int32 N);

// 出力パラメータ（複数戻り値）
UFUNCTION(BlueprintCallable)
void GetValues(int32& OutA, float& OutB);
```

---

## 関連ドキュメント

- [[Details/a_uclass]] — UCLASS の適用例・EClassFlags の対応
- [[Details/b_fproperty]] — UPROPERTY と FProperty・CPF フラグの対応
- [[Details/c_ufunction]] — UFUNCTION と UFunction・EFunctionFlags の対応
- [[Details/d_metadata]] — UMETA・meta= 指定子の詳細
