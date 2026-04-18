# ULevelStreaming / ULevelStreamingDynamic API リファレンス

- 上位: [[LevelStreaming/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Classes/Engine/LevelStreaming.h`
          `Engine/Source/Runtime/Engine/Classes/Engine/LevelStreamingDynamic.h`

---

## ULevelStreaming（基底クラス）

```cpp
UCLASS(abstract, editinlinenew, BlueprintType, Within=World, MinimalAPI)
class ULevelStreaming : public UObject
```

### BP 公開プロパティ

| プロパティ | 型 | アクセス | 説明 |
|-----------|-----|--------|------|
| `WorldAsset` | `TSoftObjectPtr<UWorld>` | 読み取り | ロード対象のワールド |
| `LevelTransform` | `FTransform` | 読み書き | レベルの配置 Transform |
| `bShouldBeLoaded` | `bool` | Setter/Getter | ロード要求フラグ |
| `bShouldBeVisible` | `bool` | Setter | 可視性フラグ |
| `bShouldBlockOnLoad` | `bool` | 読み書き | 同期ロード |
| `bShouldBlockOnUnload` | `bool` | 読み書き | 同期アンロード |
| `bDisableDistanceStreaming` | `bool` | 読み書き | 距離自動制御を無効化 |
| `LevelColor` | `FLinearColor` | — | デバッグ表示カラー |

### BP 公開メソッド

| メソッド | 説明 |
|---------|------|
| `SetShouldBeLoaded(bool)` | ロード要求フラグ設定 |
| `SetShouldBeVisible(bool)` | 可視性フラグ設定 |
| `SetPriority(int32)` | ロード優先度設定 |
| `GetLevelStreamingState() const` | 現在の状態取得 (`ELevelStreamingState`) |
| `GetIsRequestingUnloadAndRemoval() const` | アンロード・削除要求フラグ取得 |
| `SetIsRequestingUnloadAndRemoval(bool)` | アンロード・削除要求設定 |
| `HasLoadedLevel() const` | ロード済みレベルが存在するか |
| `HasLoadRequestPending() const` | ロードリクエスト保留中か |

### BP アサイン可能デリゲート

| デリゲート | タイミング |
|----------|---------|
| `OnLevelLoaded` | パッケージロード完了 |
| `OnLevelUnloaded` | アンロード完了 |
| `OnLevelShown` | AddToWorld 完了（表示） |
| `OnLevelHidden` | RemoveFromWorld 完了（非表示） |

---

## ELevelStreamingState

```cpp
enum class ELevelStreamingState : uint8
{
    Removed,          // 除去済み
    Unloaded,         // アンロード
    FailedToLoad,     // ロード失敗
    Loading,          // ロード中
    LoadedNotVisible, // ロード済み・非表示
    MakingVisible,    // 表示処理中
    LoadedVisible,    // ロード済み・表示中
    MakingInvisible,  // 非表示処理中
};
```

---

## ULevelStreamingDynamic

```cpp
UCLASS(BlueprintType, MinimalAPI)
class ULevelStreamingDynamic : public ULevelStreaming
```

### BP 公開静的メソッド

| メソッド | 説明 |
|---------|------|
| `LoadLevelInstance(WorldCtx, LevelName, Location, Rotation, bOutSuccess, ...)` | 名前でレベルをインスタンス化 |
| `LoadLevelInstanceBySoftObjectPtr(WorldCtx, Level, Location, Rotation, bOutSuccess, ...)` | ソフト参照でレベルをインスタンス化 |

### FLoadLevelInstanceParams（C++ 詳細制御）

```cpp
struct FLoadLevelInstanceParams
{
    UWorld* World;
    FString LongPackageName;
    FTransform LevelTransform;
    const FString* OptionalLevelNameOverride;         // ネットワーク同期用
    TSubclassOf<ULevelStreamingDynamic> OptionalLevelStreamingClass;
    bool bLoadAsTempPackage;      // /Temp プレフィックス
    bool bInitiallyVisible;       // 初期表示
    bool bAllowReuseExitingLevelStreaming; // 既存の再利用
    TUniqueFunction<void(ULevelStreaming*)> LevelStreamingCreatedCallback;
};

// 使用例
ULevelStreamingDynamic* LS = ULevelStreamingDynamic::LoadLevelInstance(Params, bSuccess);
```

---

## ALevelStreamingVolume

```cpp
UCLASS(MinimalAPI, HideCategories=(Brush, Physics, Input, ...))
class ALevelStreamingVolume : public AVolume
```

| プロパティ | 型 | 説明 |
|-----------|-----|------|
| `StreamingLevels` | `TArray<ULevelStreaming*>` | 制御対象のストリーミングオブジェクト |
| `bDisabled` | `bool` | ボリュームを無効化 |
| `StreamingUsage` | `EStreamingVolumeUsage` | 用途（ロードのみ / 可視性 / 両方） |

---

## FLevelStreamingDelegates（グローバルデリゲート）

```cpp
class FLevelStreamingDelegates
{
public:
    // 全ストリーミングレベルへのデリゲート（ULevelStreaming の方は個別）
    static ENGINE_API FOnLevelStreamingStateChanged& OnLevelStreamingStateChanged();
    static ENGINE_API FOnLevelBeginMakingVisible& OnLevelBeginMakingVisible();
    static ENGINE_API FOnLevelBeginMakingInvisible& OnLevelBeginMakingInvisible();
};
```

---

## ULevelStreamingAlwaysLoaded / ULevelStreamingPersistent

```cpp
// 常時ロード（Sublevel として常に読み込まれる）
UCLASS(MinimalAPI)
class ULevelStreamingAlwaysLoaded : public ULevelStreaming { ... };

// PersistentLevel 用の内部クラス
UCLASS(MinimalAPI)
class ULevelStreamingPersistent : public ULevelStreaming { ... };
```

`ULevelStreamingAlwaysLoaded` は距離やボリュームに関係なく常時ロードされる。`SetShouldBeLoaded(false)` を呼んでも無視される。

---

## BP レシピ: 動的ロードと状態監視

```
[Button Pressed]
  ↓
[LoadLevelInstance]
  → WorldContext: Self
  → LevelName: "/Game/Levels/ShopLevel"
  → Location: (1000, 0, 0)
  → Rotation: (0, 0, 0)
  → SuccessOut: (bool)
  → Return: ULevelStreamingDynamic

[Bind Event to OnLevelShown]
  → Target: ULevelStreamingDynamic
  → Event: OnShopLevelReady

[OnShopLevelReady]
  → Show UI (ショップ画面を開く)

[Button Dismissed]
  ↓
[Set Should Be Visible] → false
[Set Should Be Loaded] → false
[Set Is Requesting Unload And Removal] → true
```
