# UWorldPartition API リファレンス

- 上位: [[WorldPartition/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Public/WorldPartition/WorldPartition.h`
          `Engine/Source/Runtime/Engine/Public/WorldPartition/WorldPartitionSubsystem.h`
          `Engine/Source/Runtime/Engine/Public/WorldPartition/WorldPartitionBlueprintLibrary.h`

---

## UWorldPartition

WP の中核クラス。`UWorld` から `GetWorldPartition()` で取得。**直接 BP 公開なし**。

### 初期化・状態

| メソッド | 説明 |
|---------|------|
| `Initialize(UWorld*, const FTransform&)` | WP 初期化（エンジン内部から呼び出し） |
| `Uninitialize()` | WP 解放 |
| `IsInitialized() const` | 初期化済みか |
| `IsStreamingEnabled() const` | ストリーミングが有効か |
| `CanStream() const` | ストリーミング実行可能か |

### ストリーミングソース

| メソッド | 説明 |
|---------|------|
| `RegisterStreamingSourceProvider(IWorldPartitionStreamingSourceProvider*)` | ソースプロバイダ登録 |
| `UnregisterStreamingSourceProvider(IWorldPartitionStreamingSourceProvider*)` | ソース登録解除 |
| `GetStreamingSources(TArray<FWorldPartitionStreamingSource>&) const` | 現在のソース一覧取得 |

### セル・ストリーミング制御

| メソッド | 説明 |
|---------|------|
| `IsStreamingCompleted(EStreamingSourceTargetState, const TArray<FWorldPartitionStreamingQuerySource>&, bool) const` | 指定ソース範囲のストリーミングが完了しているか |
| `GetStreamingCells(const FWorldPartitionStreamingQuerySource&, TSet<const UWorldPartitionRuntimeCell*>&) const` | クエリにマッチするセル取得 |
| `DrawRuntimeHash2D(FWorldPartitionDraw2DContext&)` | 2D デバッグ描画 |

### エディタ専用 (WITH_EDITOR)

| メソッド | 説明 |
|---------|------|
| `LoadAllActors(bool)` | 全アクタをロード（エディタ内） |
| `UnloadAllActors(bool)` | 全アクタをアンロード |
| `GetActorEditorContextClient()` | エディタコンテキスト取得 |

---

## UWorldPartitionSubsystem

ゲームインスタンスサブシステム。BP から `GetGameInstanceSubsystem(UWorldPartitionSubsystem)` で取得可能。

### BP公開メソッド

```cpp
// ストリーミング完了確認
UFUNCTION(BlueprintCallable, Category = "WorldPartition")
bool IsStreamingCompleted(
    AActor* Actor,
    EStreamingSourceTargetState TargetState,
    bool bExactState) const;

// 全 WP 取得
UFUNCTION(BlueprintCallable, Category = "WorldPartition")
void GetWorldPartitions(TArray<UWorldPartition*>& OutWorldPartitions) const;
```

---

## UWorldPartitionBlueprintLibrary

静的 BP ライブラリ。どこからでも呼び出せる。

### 主要関数

```cpp
// ストリーミング完了待ち
UFUNCTION(BlueprintCallable, Category = "WorldPartition")
static bool IsStreamingCompleted(
    const UObject* WorldContextObject,
    const TArray<FWorldPartitionStreamingQuerySource>& Sources);

// DataLayer 操作
UFUNCTION(BlueprintCallable, Category = "WorldPartition")
static bool SetDataLayerInstanceRuntimeState(
    const UObject* WorldContextObject,
    const UDataLayerAsset* DataLayerAsset,
    EDataLayerRuntimeState State);

// ストリーミングソースコンポーネントの取得
UFUNCTION(BlueprintCallable, Category = "WorldPartition")
static UWorldPartitionStreamingSourceComponent* GetStreamingSource(
    AActor* Actor);
```

---

## FWorldPartitionStreamingQuerySource（BP 公開）

```cpp
USTRUCT(BlueprintType)
struct FWorldPartitionStreamingQuerySource
{
    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = "Query")
    FVector Location;           // クエリ位置

    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = "Query")
    float Radius;               // クエリ半径

    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = "Query")
    bool bUseGridLoadingRange;  // グリッドのロード距離を使用

    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = "Query")
    TArray<FName> DataLayers;   // 対象 DataLayer（空 = 非 DL セルのみ）

    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = "Query")
    bool bDataLayersOnly;       // DL セルのみ対象

    UPROPERTY(BlueprintReadWrite, EditAnywhere, Category = "Query")
    bool bSpatialQuery;         // 空間クエリ（false = AlwaysLoaded のみ）
};
```

---

## IWorldPartitionStreamingSourceProvider（インターフェース）

カスタムストリーミングソースを提供するインターフェース。`APlayerController` が実装済み。

```cpp
class IWorldPartitionStreamingSourceProvider
{
public:
    // 単一ソースを提供
    virtual bool GetStreamingSource(FWorldPartitionStreamingSource& StreamingSource) const;

    // 複数ソースを提供（デフォルト実装は GetStreamingSource を呼ぶ）
    virtual bool GetStreamingSources(TArray<FWorldPartitionStreamingSource>& StreamingSources) const;
};
```

カスタム実装例：

```cpp
// ActorComponent で IWorldPartitionStreamingSourceProvider を実装
UCLASS()
class UMyStreamingSourceComponent : public UActorComponent,
                                    public IWorldPartitionStreamingSourceProvider
{
    GENERATED_BODY()

    virtual bool GetStreamingSource(FWorldPartitionStreamingSource& Out) const override
    {
        Out = FWorldPartitionStreamingSource(
            TEXT("MySource"),
            GetOwner()->GetActorLocation(),
            GetOwner()->GetActorRotation(),
            EStreamingSourceTargetState::Activated,
            false,
            EStreamingSourcePriority::Default,
            false
        );
        return true;
    }
};
```

---

## 関連クラス

| クラス | ファイル | 役割 |
|-------|---------|------|
| `UWorldPartitionRuntimeHashSet` | `RuntimeHashSet/WorldPartitionRuntimeHashSet.h` | ランタイムハッシュ（グリッド管理） |
| `UWorldPartitionStreamingPolicy` | `WorldPartitionStreamingPolicy.h` | ストリーミング判定ロジック |
| `UWorldPartitionStreamingSourceComponent` | `WorldPartitionStreamingSourceComponent.h` | カスタムソースコンポーネント |
| `UDataLayerSubsystem` | `DataLayer/DataLayerSubsystem.h` | DataLayer 状態管理 |
