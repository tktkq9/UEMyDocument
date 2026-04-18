# UWorldPartitionRuntimeHashSet / URuntimePartition API リファレンス

- 上位: [[WorldPartition/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Public/WorldPartition/RuntimeHashSet/WorldPartitionRuntimeHashSet.h`
          `Engine/Source/Runtime/Engine/Public/WorldPartition/RuntimeHashSet/RuntimePartition.h`
          `Engine/Source/Runtime/Engine/Public/WorldPartition/RuntimeHashSet/RuntimePartitionLHGrid.h`

---

## UWorldPartitionRuntimeHash（基底）

```cpp
UCLASS(Abstract, MinimalAPI)
class UWorldPartitionRuntimeHash : public UObject
{
public:
    // ストリーミング生成（クック・PIE 開始時）
    virtual bool GenerateStreaming(...) PURE_VIRTUAL(...);

    // 全セルの取得
    virtual void GetAllStreamingCells(TSet<const UWorldPartitionRuntimeCell*>&, ...) const PURE_VIRTUAL(...);

    // ストリーミングソースに基づくセル取得
    virtual bool GetStreamingCells(
        const FWorldPartitionStreamingQuerySource&,
        FWorldPartitionRuntimeCellsStreamingSourceInfo&) const PURE_VIRTUAL(...);
};
```

---

## UWorldPartitionRuntimeHashSet（メイン実装）

```cpp
UCLASS(MinimalAPI)
class UWorldPartitionRuntimeHashSet : public UWorldPartitionRuntimeHash
{
    // エディタで設定するパーティション定義
    UPROPERTY(EditAnywhere, Category = RuntimeSettings)
    TArray<FRuntimePartitionDesc> RuntimePartitions;

    // クック後にシリアライズされるランタイムデータ
    UPROPERTY()
    TArray<FRuntimePartitionStreamingData> StreamingData;
};
```

### メソッド

| メソッド | 説明 |
|---------|------|
| `GenerateStreaming(...)` | エディタ時に全アクタをセルに割り当て |
| `GetStreamingCells(...)` | ストリーミングソースからアクティブセルを取得 |
| `GetAllStreamingCells(...)` | 全セルを取得（デバッグ・クック用） |
| `IsCellRelevantFor(...)` | セルがソースに関連するか判定 |

---

## FRuntimePartitionDesc（USTRUCT）

```cpp
USTRUCT()
struct FRuntimePartitionDesc
{
    // パーティション識別名（アクタの RuntimeGrid で参照）
    UPROPERTY(EditAnywhere, Category = RuntimeSettings)
    FName Name;

    // メインパーティション（グリッド分割を担当）
    UPROPERTY(EditAnywhere, Category = RuntimeSettings, Instanced)
    TObjectPtr<URuntimePartition> MainLayer;

    // HLOD 用パーティション設定（階層ごとに 1 つ）
    UPROPERTY(EditAnywhere, Category = RuntimeSettings)
    TArray<FRuntimePartitionHLODSetup> HLODSetups;
};
```

---

## URuntimePartition（基底）

```cpp
UCLASS(Abstract, MinimalAPI, EditInlineNew)
class URuntimePartition : public UObject
{
public:
    // クック時のストリーミング生成
    virtual bool GenerateStreaming(
        const FGenerateStreamingParams& InParams,
        FGenerateStreamingResult& OutResult) PURE_VIRTUAL(...);

    // ロード距離取得
    virtual int32 GetLoadingRange() const PURE_VIRTUAL(...);

    // このパーティションが HLOD をサポートするか
    virtual bool SupportsHLODs() const { return false; }
};
```

---

## URuntimePartitionLHGrid（LH グリッド）

最も一般的な実装。ワールドを均一グリッドに分割。

```cpp
UCLASS(MinimalAPI)
class URuntimePartitionLHGrid : public URuntimePartition
{
#if WITH_EDITORONLY_DATA
    // セルサイズ（cm）。デフォルト 25600 = 256m
    UPROPERTY(EditAnywhere, Category = RuntimeSettings)
    uint32 CellSize = 25600;

    // グリッド原点オフセット
    UPROPERTY(EditAnywhere, Category = RuntimeSettings)
    FVector Origin = FVector::ZeroVector;

    // 2D モード（Z 軸を無視してロード判定）
    UPROPERTY(EditAnywhere, Category = RuntimeSettings)
    bool bIs2D = false;
#endif
};
```

### CellSize の計算

- ロード範囲 = `CellSize * N`（N はエディタのロードセル数設定）
- `bIs2D = true` の場合、Z 方向のセル分割は行わず 1 層扱い

---

## URuntimePartitionPersistent

常時ロードされるアクタのパーティション。`bIsAlwaysLoaded = true` のアクタが配置される。

```cpp
UCLASS(MinimalAPI)
class URuntimePartitionPersistent : public URuntimePartition
{
    // ロード距離なし（常にロード）
    virtual int32 GetLoadingRange() const override { return 0; }
};
```

---

## FRuntimePartitionStreamingData（クック済みデータ）

クック後にシリアライズされ、ランタイムで空間インデックスを構築。

```cpp
USTRUCT()
struct FRuntimePartitionStreamingData
{
    FName Name;         // パーティション名
    int32 LoadingRange; // ロード距離（cm）

    // 空間的にロードされるセル（距離に応じてロード）
    TArray<TObjectPtr<UWorldPartitionRuntimeCell>> SpatiallyLoadedCells;

    // 常時ロードセル（距離によらずロード）
    TArray<TObjectPtr<UWorldPartitionRuntimeCell>> NonSpatiallyLoadedCells;

    // ランタイムに構築される Hilbert R-tree（Transient）
    mutable TUniquePtr<FStaticSpatialIndexType>   SpatialIndex;    // 3D
    mutable TUniquePtr<FStaticSpatialIndexType2D> SpatialIndex2D;  // 2D 投影
    mutable TUniquePtr<FStaticSpatialIndexType2D> SpatialIndexForce2D; // 強制 2D

    // インデックス構築・解放
    void CreatePartitionsSpatialIndex() const;
    void DestroyPartitionsSpatialIndex() const;

    // ロード距離取得
    int32 GetLoadingRange() const;
};
```

---

## 関連クラス

| クラス | ファイル | 役割 |
|-------|---------|------|
| `URuntimePartitionLevelStreaming` | `RuntimePartitionLevelStreaming.h` | レベルストリーミング方式パーティション |
| `FStaticSpatialIndex` | `StaticSpatialIndex.h` | R-tree 空間インデックス基盤 |
| `UWorldPartitionRuntimeCell` | `WorldPartitionRuntimeCell.h` | セル単位のストリーミング制御 |
| `FWorldPartitionStreamingSource` | `WorldPartitionStreamingSource.h` | ストリーミング基準点 |
