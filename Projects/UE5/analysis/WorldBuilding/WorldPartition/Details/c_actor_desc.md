# WorldPartition ActorDesc・エディタ連携

- 上位: [[WorldPartition/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Public/WorldPartition/WorldPartitionActorDesc.h`
          `Engine/Source/Runtime/Engine/Public/WorldPartition/ActorDescContainerInstance.h`

---

## 概要

**ActorDesc（アクタ記述子）** は、アクタをメモリにロードせずにそのメタデータ（バウンド・DataLayer・クラス等）を参照するためのエディタ専用構造体。World Partition がオープンワールドのアクタを全ロードせずにストリーミング計画を立てられるのはこの仕組みのおかげ。

> `WITH_EDITOR` ガードで囲まれており、**ランタイムビルドには含まれない**。

---

## クラス構造

```mermaid
classDiagram
    class FWorldPartitionActorDesc {
        +Guid: FGuid
        +ActorPackage: FName
        +ActorLabel: FName
        +RuntimeGrid: FName
        +bIsSpatiallyLoaded: bool
        +GetEditorBounds() FBox
        +GetRuntimeBounds() FBox
        +GetDataLayers() TArray~FName~
    }
    class FWorldPartitionActorDescInstance {
        +ContainerInstance: UActorDescContainerInstance*
        +GetDataLayerInstanceNames() TArray~FName~
        +IsRuntimeRelevant() bool
    }
    class UActorDescContainer {
        +ActorDescs: TArray~FWorldPartitionActorDesc~
        +GetActorDescs()
    }
    class UActorDescContainerInstance {
        +Container: UActorDescContainer*
        +GetActorDescInstance(Guid)
    }

    FWorldPartitionActorDescInstance --> FWorldPartitionActorDesc : wraps
    UActorDescContainerInstance --> UActorDescContainer : refs
    UActorDescContainer --> FWorldPartitionActorDesc : stores
```

---

## FWorldPartitionActorDesc — 主要フィールド

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `Guid` | `FGuid` | アクタの一意識別子 |
| `ActorPackage` | `FName` | アクタのパッケージ名（ディスク上） |
| `ActorLabel` | `FName` | エディタ表示名 |
| `RuntimeGrid` | `FName` | 所属パーティション名 |
| `bIsSpatiallyLoaded` | `bool` | 空間的にロードされるか |
| `bActorIsEditorOnly` | `bool` | エディタ専用アクタか |
| `HLODLayer` | `FSoftObjectPath` | 対応する HLOD レイヤー |
| `ActorTransform` | `FTransform` | ワールド変換 |

### 主要メソッド

```cpp
// バウンド取得
FBox GetEditorBounds() const;   // エディタ用（コリジョン含む）
FBox GetRuntimeBounds() const;  // ランタイム用（ストリーミング判定）

// DataLayer 取得（UE5.4 以降）
TArray<FName> GetDataLayers(bool bIncludeExternalDataLayer = true) const;

// メタデータ
FName GetRuntimeGrid() const;
bool GetIsSpatiallyLoaded() const;
bool GetActorIsHLODRelevant() const;
```

---

## ActorDesc の生成・シリアライズ

アクタが WP ワールドに保存されるとき、`AActor::CreateClassActorDesc()` がオーバーライドされて `FWorldPartitionActorDesc` が生成される。シリアライズされたデータはパッケージヘッダに埋め込まれ、エディタ起動時に全パッケージをスキャンして読み込まれる。

```mermaid
sequenceDiagram
    participant Editor as UE Editor
    participant WP as UWorldPartition
    participant Container as UActorDescContainer
    participant Desc as FWorldPartitionActorDesc

    Editor->>WP: Initialize（ワールドロード時）
    WP->>Container: LoadAllActorDescs()
    Container->>Container: パッケージヘッダをスキャン
    Container->>Desc: Deserialize（アクタをロードせず）
    Desc-->>Container: メタデータのみキャッシュ
    WP->>WP: ストリーミング計画に利用
```

---

## FWorldPartitionActorDescInstance — ランタイム解決

`FWorldPartitionActorDesc` は純粋なデータ。`FWorldPartitionActorDescInstance` がコンテナインスタンスのコンテキストで実際のランタイム状態（DataLayerInstance 名前解決等）を付加する。

```cpp
class FWorldPartitionActorDescInstance
{
public:
    // DataLayerInstance 名の解決済みリスト（ランタイム）
    bool HasResolvedDataLayerInstanceNames() const;
    TArray<FName> GetDataLayerInstanceNames() const;

    // ランタイム関連性の判定
    bool IsRuntimeRelevant(const FActorContainerID& InContainerID) const;
    bool IsEditorRelevant() const;

    // コンテナ情報
    UActorDescContainerInstance* GetContainerInstance() const;
    const FTransform& GetContainerTransform() const;
};
```

---

## FWorldPartitionRelativeBounds — 相対バウンド

アクタの位置をコンテナの Transform からの相対座標で保持する。Level Instance（入れ子ワールド）での再利用に対応。

```cpp
struct FWorldPartitionRelativeBounds
{
    FVector Center;
    FQuat Rotation;
    FVector Extents;
    bool bIsValid;

    FBox ToAABB() const;  // AABB に変換
    FWorldPartitionRelativeBounds TransformBy(const FTransform&) const;
};
```

---

## アクタ側での登録

`AActor` にはいくつかの WP 関連プロパティがある（BP でも設定可）。

```cpp
// AActor のプロパティ
UPROPERTY(EditAnywhere, Category = "World Partition")
FName RuntimeGrid;           // 所属パーティション

UPROPERTY(EditAnywhere, Category = "World Partition")
bool bIsSpatiallyLoaded;     // 空間ロードか AlwaysLoaded か

UPROPERTY(EditAnywhere, Category = "World Partition")
bool bActorIsHLODRelevant;   // HLOD 生成に含めるか

UPROPERTY(EditAnywhere, Category = "World Partition")
TObjectPtr<UHLODLayer> HLODLayer; // 使用する HLOD レイヤー
```

---

## エディタでの利用

- **World Partition Editor**（Window → World Partition）でセル・アクタ分布を可視化
- **ミニマップ**は ActorDesc のバウンドを使ってアクタ位置をプレビュー
- クック時は ActorDesc からセル→アクタのマッピングを生成し、実際のパッケージ参照を解決する
