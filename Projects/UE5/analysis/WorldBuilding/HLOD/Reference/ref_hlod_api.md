# UWorldPartitionHLOD / UHLODLayer API リファレンス

- 上位: [[HLOD/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Public/WorldPartition/HLOD/`

---

## AWorldPartitionHLOD

```cpp
UCLASS(NotPlaceable, MinimalAPI,
       HideCategories=(Rendering, Replication, Collision, Physics,
                       Navigation, Networking, Input, Actor, LevelInstance, Cooking))
class AWorldPartitionHLOD : public AActor, public IWorldPartitionHLODObject
```

### プロパティ

| プロパティ | 型 | 説明 |
|-----------|-----|------|
| `LODLevel` | `uint32` | HLOD 階層レベル（0 = 最詳細、数が大きいほど遠距離用） |
| `bRequireWarmup` | `bool` | GPU ウォームアップが必要か（Nanite 等） |

### メソッド

| メソッド | BP公開 | 説明 |
|---------|--------|------|
| `GetLODLevel() const` | No | LOD レベル取得 |
| `SetVisibility(bool)` | No | 可視性設定（IWorldPartitionHLODObject 実装） |
| `GetSourceCellGuid() const` | No | 対応するセルの GUID |
| `IsStandalone() const` | No | セルに紐づかない独立 HLOD か |
| `IsCustomHLOD() const` | No | カスタム HLOD か |
| `GetAssetsToWarmup() const` | No | ウォームアップ対象アセット一覧 |

---

## UHLODLayer

```cpp
UCLASS(Blueprintable, MinimalAPI)
class UHLODLayer : public UObject
```

### エディタ専用メソッド (WITH_EDITOR)

| メソッド | 説明 |
|---------|------|
| `GetEngineDefaultHLODLayersSetup()` | プロジェクトデフォルト HLOD レイヤー取得（static） |
| `DuplicateHLODLayersSetup(UHLODLayer*, FString, FString)` | レイヤー設定の複製（static） |
| `GetLayerType() const` | `EHLODLayerType` 取得 |
| `GetHLODBuilderClass() const` | ビルダークラス取得 |
| `GetHLODBuilderSettings() const` | ビルダー設定取得 |
| `GetParentLayer() const` | 親レイヤー取得 |
| `SetParentLayer(UHLODLayer*)` | 親レイヤー設定 |
| `DoesRequireWarmup() const` | ウォームアップ要否 |
| `ComputeHLODHash(FHLODHashBuilder&)` | ハッシュ計算（変更検出用） |

---

## UWorldPartitionHLODRuntimeSubsystem

`UWorldSubsystem` の派生。`GetWorld()->GetSubsystem<UWorldPartitionHLODRuntimeSubsystem>()` で取得。

### メソッド

| メソッド | 説明 |
|---------|------|
| `RegisterHLODObject(IWorldPartitionHLODObject*)` | HLOD オブジェクト登録 |
| `UnregisterHLODObject(IWorldPartitionHLODObject*)` | HLOD オブジェクト登録解除 |
| `OnCellShown(const UWorldPartitionRuntimeCell*)` | セル表示時コールバック |
| `OnCellHidden(const UWorldPartitionRuntimeCell*)` | セル非表示時コールバック |
| `CanMakeVisible(const UWorldPartitionRuntimeCell*)` | 元セルを表示可能か判定 |
| `CanMakeInvisible(const UWorldPartitionRuntimeCell*)` | 元セルを非表示可能か判定 |
| `GetHLODObjectsForCell(const UWorldPartitionRuntimeCell*)` | セルの HLOD 一覧取得 |
| `IsHLODEnabled()` | HLOD が有効か（static） |

### デリゲート

```cpp
// HLOD 登録・解除イベント
FWorldPartitionHLODObjectRegisteredEvent&   OnHLODObjectRegisteredEvent();
FWorldPartitionHLODObjectUnregisteredEvent& OnHLODObjectUnregisteredEvent();

// セル内の全 HLOD を走査
FWorldPartitionHLODForEachHLODObjectInCellEvent& GetForEachHLODObjectInCellEvent();
```

---

## UHLODBuilder（カスタム実装用）

```cpp
UCLASS(Abstract, MinimalAPI)
class UHLODBuilder : public UObject
{
public:
    // ソースコンポーネントから HLOD コンポーネントを生成（純粋仮想）
    virtual TArray<UActorComponent*> Build(
        const FHLODBuildContext& InHLODBuildContext,
        const TArray<UActorComponent*>& InSourceComponents) const PURE_VIRTUAL(...);
};
```

### FHLODBuildContext フィールド

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `World` | `UWorld*` | 対象ワールド |
| `SourceComponents` | `TArray<UActorComponent*>` | まとめ対象コンポーネント |
| `AssetsOuter` | `UObject*` | 生成アセットの Outer |
| `AssetsBaseName` | `FString` | 生成アセットのベース名 |
| `WorldPosition` | `FVector` | HLOD アクタの位置 |
| `MinVisibleDistance` | `double` | 最小表示距離（cm） |

---

## カスタム HLOD ビルダーの実装例

```cpp
UCLASS()
class UMyHLODBuilder : public UHLODBuilder
{
    GENERATED_BODY()

public:
    virtual TArray<UActorComponent*> Build(
        const FHLODBuildContext& Context,
        const TArray<UActorComponent*>& SourceComponents) const override
    {
        // カスタム HLOD メッシュ生成処理
        UStaticMeshComponent* HLODComp = NewObject<UStaticMeshComponent>(
            Context.AssetsOuter);
        // ... メッシュ生成 ...
        return { HLODComp };
    }
};
```

---

## 関連ファイル

| ファイル | クラス | 説明 |
|---------|-------|------|
| `HLODActor.h` | `AWorldPartitionHLOD` | HLOD アクタ本体 |
| `HLODActorDesc.h` | `FHLODActorDesc` | HLOD アクタ記述子（エディタ） |
| `HLODBuilder.h` | `UHLODBuilder` / `UHLODBuilderSettings` | ビルド処理 |
| `HLODLayer.h` | `UHLODLayer` | HLOD 設定アセット |
| `HLODRuntimeSubsystem.h` | `UWorldPartitionHLODRuntimeSubsystem` | ランタイム管理 |
| `HLODSubsystem.h` | — | 旧名 `UHLODSubsystem`（UE5.4 で deprecated） |
| `HLODStats.h` | `FHLODStats` | ビルド統計情報 |
