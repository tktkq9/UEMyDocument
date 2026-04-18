# UDataLayerAsset / UDataLayerManager API リファレンス

- 上位: [[DataLayer/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Public/WorldPartition/DataLayer/`

---

## UDataLayerAsset

```cpp
UCLASS(BlueprintType, editinlinenew, MinimalAPI)
class UDataLayerAsset : public UDataAsset
```

### BP 公開メソッド

| メソッド | 説明 |
|---------|------|
| `GetType() const` | `EDataLayerType` を取得（Runtime / Editor） |
| `IsRuntime() const` | Runtime タイプかつ Private でないか |
| `GetDebugColor() const` | デバッグ表示カラーを取得 |
| `IsClientOnly() const` | ClientOnly フィルタか |
| `IsServerOnly() const` | ServerOnly フィルタか |

---

## UDataLayerManager

```cpp
UCLASS(Config = Engine, Within = WorldPartition, MinimalAPI)
class UDataLayerManager : public UObject
```

### 取得方法

```cpp
// 推奨（UE5.3+）
UDataLayerManager* DLM = UDataLayerManager::GetDataLayerManager(this);

// テンプレート版（任意の型から）
template <class T>
static UDataLayerManager* GetDataLayerManager(const T* InObject);
```

### BP 公開メソッド

| メソッド | 説明 |
|---------|------|
| `GetDataLayerInstances() const` | 全インスタンスを取得 |
| `GetDataLayerInstanceFromAsset(UDataLayerAsset*) const` | アセットからインスタンスを取得 |
| `GetDataLayerInstanceFromName(FName) const` | 名前からインスタンスを取得 |
| `SetDataLayerInstanceRuntimeState(UDataLayerInstance*, State, bRecursive)` | インスタンス指定で状態変更 |
| `SetDataLayerRuntimeState(UDataLayerAsset*, State, bRecursive)` | アセット指定で状態変更（ショートカット） |
| `GetDataLayerInstanceRuntimeState(UDataLayerInstance*) const` | 要求ランタイム状態を取得 |
| `GetDataLayerInstanceEffectiveRuntimeState(UDataLayerInstance*) const` | 実効ランタイム状態を取得 |

### C++ 専用メソッド

| メソッド | 説明 |
|---------|------|
| `ForEachDataLayerInstance(TFunctionRef<bool(UDataLayerInstance*)>)` | 全インスタンスを走査 |
| `GetEffectiveActiveDataLayerNames() const` | Activated 状態のレイヤー名 Set |
| `GetEffectiveLoadedDataLayerNames() const` | Loaded 以上のレイヤー名 Set |
| `IsAnyDataLayerInEffectiveRuntimeState(TArray<FName>, State) const` | 指定レイヤーがいずれか指定状態か |

### デリゲート

```cpp
// 状態変更イベント（BP アサイン可能）
UPROPERTY(BlueprintAssignable)
FOnDataLayerInstanceRuntimeStateChanged OnDataLayerInstanceRuntimeStateChanged;

// シグネチャ
DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(
    FOnDataLayerInstanceRuntimeStateChanged,
    const UDataLayerInstance*, DataLayer,
    EDataLayerRuntimeState, State);
```

---

## UDataLayerInstance

```cpp
UCLASS(Config = Engine, PerObjectConfig, BlueprintType, MinimalAPI)
class UDataLayerInstance : public UObject
```

### BP 公開メソッド

| メソッド | 説明 |
|---------|------|
| `GetDataLayerFName() const` | レイヤーの識別名 |
| `IsRuntime() const` | Runtime タイプか |
| `GetInitialRuntimeState() const` | 初期ランタイム状態を取得 |
| `GetOuterWorld() const` | 所属ワールドを取得 |

### エディタ専用メソッド (WITH_EDITOR)

| メソッド | 説明 |
|---------|------|
| `SetParent(UDataLayerInstance*)` | 親レイヤー設定 |
| `SetVisible(bool)` | エディタ可視性切り替え |
| `SetIsLoadedInEditor(bool, bool)` | エディタでのロード状態設定 |
| `SetInitialRuntimeState(EDataLayerRuntimeState)` | 初期状態設定 |

---

## EDataLayerRuntimeState（再掲）

```cpp
UENUM(BlueprintType)
enum class EDataLayerRuntimeState : uint8
{
    Unloaded,   // アンロード（メモリになし）
    Loaded,     // ロード済み（不可視）
    Activated,  // ロード＋表示（BeginPlay 済み）
};
```

---

## UExternalDataLayerAsset

```cpp
class UExternalDataLayerAsset : public UDataLayerAsset
```

| メソッド | 説明 |
|---------|------|
| `GetExternalActorFolder() const` | 外部アクタの格納フォルダパス |

---

## AWorldDataLayers

```cpp
UCLASS(NotPlaceable, MinimalAPI)
class AWorldDataLayers : public AInfo
```

| メソッド | 説明 |
|---------|------|
| `Get(const UWorld*)` | ワールドの AWorldDataLayers を取得（static） |
| `GetDataLayerInstances() const` | 全インスタンスを取得 |
| `AddDataLayer(UDataLayerAsset*)` | レイヤー追加（エディタ） |
| `RemoveDataLayer(UDataLayerInstance*)` | レイヤー削除（エディタ） |

---

## 使用例（C++）

```cpp
// DataLayer の状態を Activated に変更
void AMyGameMode::ActivateNightMode()
{
    UDataLayerManager* DLM = UDataLayerManager::GetDataLayerManager(this);
    if (!DLM) return;

    if (UDataLayerAsset* NightLayer = NightDataLayerAsset.Get())
    {
        DLM->SetDataLayerRuntimeState(NightLayer,
            EDataLayerRuntimeState::Activated,
            /*bRecursive=*/true);
    }
}

// 状態変更イベントをバインド
void AMyGameMode::BeginPlay()
{
    Super::BeginPlay();

    if (UDataLayerManager* DLM = UDataLayerManager::GetDataLayerManager(this))
    {
        DLM->OnDataLayerInstanceRuntimeStateChanged.AddDynamic(
            this, &AMyGameMode::OnDataLayerStateChanged);
    }
}
```
