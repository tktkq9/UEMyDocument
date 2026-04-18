# HLOD 生成・AWorldPartitionHLOD・MeshMerge/Imposter

- 上位: [[HLOD/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Public/WorldPartition/HLOD/HLODActor.h`
          `Engine/Source/Runtime/Engine/Public/WorldPartition/HLOD/HLODBuilder.h`

---

## 概要

World Partition HLOD は、遠距離で表示するための**低詳細度の代替メッシュ**をビルド時に自動生成する。カメラが遠ざかると元のアクタを非表示にして HLOD アクタを表示し、描画コストを削減する。

---

## クラス構造

```mermaid
classDiagram
    class AWorldPartitionHLOD {
        +LODLevel: uint32
        +bRequireWarmup: bool
        +SetVisibility(bool)
        +GetSourceCellGuid() FGuid
        +IsStandalone() bool
    }
    class UHLODLayer {
        +LayerType: EHLODLayerType
        +HLODBuilderClass: TSubclassOf~UHLODBuilder~
        +HLODBuilderSettings: UHLODBuilderSettings*
        +ParentLayer: UHLODLayer*
    }
    class UHLODBuilder {
        <<abstract>>
        +Build(FHLODBuildContext) TArray~UActorComponent~*
    }
    class UHLODBuilderSettings {
        <<abstract>>
        +ComputeHLODHash(FHLODHashBuilder&)
    }

    AWorldPartitionHLOD --> UHLODLayer : uses
    UHLODLayer --> UHLODBuilder : references class
    UHLODLayer --> UHLODBuilderSettings : instanced
    UHLODBuilder --> UHLODBuilderSettings : reads
```

---

## EHLODLayerType — HLOD 生成方式

```cpp
enum class EHLODLayerType : uint8
{
    Instancing,       // インスタンシング（ISM でまとめて描画）
    MeshMerge,        // メッシュ結合（単一スタティックメッシュ）
    MeshSimplify,     // メッシュ簡略化（ポリゴン削減）
    MeshApproximate,  // メッシュ近似（ボリュームベース近似）
    Custom,           // カスタムビルダー（UHLODBuilder の派生クラスを指定）
    CustomHLODActor,  // カスタム HLOD アクタ
};
```

| 方式 | 特徴 | 推奨シーン |
|------|------|---------|
| Instancing | 高速・元マテリアル保持 | 繰り返し配置のオブジェクト（木・岩等） |
| MeshMerge | 単一ドローコール | 建物・構造物の近距離 HLOD |
| MeshSimplify | ポリゴン削減 | 中程度の距離 |
| MeshApproximate | ビルド速度重視の近似 | 遠距離・広域 |
| Custom | 完全カスタマイズ | 特殊要件 |

---

## AWorldPartitionHLOD

```cpp
UCLASS(NotPlaceable, MinimalAPI)
class AWorldPartitionHLOD : public AActor, public IWorldPartitionHLODObject
{
    // HLOD 階層レベル（0 = アクタを直接まとめた HLOD、1 = HLOD0 をまとめた HLOD）
    UPROPERTY()
    uint32 LODLevel;

    // ウォームアップ（Nanite 等の事前ロード）が必要か
    bool bRequireWarmup;

    // 表示切り替え
    virtual void SetVisibility(bool bIsVisible) override;

    // 元セルの GUID（どのセルを代替しているか）
    virtual const FGuid& GetSourceCellGuid() const override;

    // スタンドアロン HLOD（セルに紐づかない独立した HLOD）
    virtual bool IsStandalone() const override;
};
```

---

## UHLODLayer — HLOD レイヤー設定アセット

**Content Browser** で作成するデータアセット（`.uasset`）。各アクタの `HLODLayer` プロパティで参照する。

```cpp
UCLASS(Blueprintable, MinimalAPI)
class UHLODLayer : public UObject
{
    // 生成方式
    EHLODLayerType LayerType;

    // カスタムビルダークラス（LayerType == Custom 時）
    TSubclassOf<UHLODBuilder> HLODBuilderClass;

    // ビルダー固有の設定
    TObjectPtr<UHLODBuilderSettings> HLODBuilderSettings;

    // 親レイヤー（HLOD 階層の次レベル）
    TObjectPtr<UHLODLayer> ParentLayer;
};
```

### ParentLayer による階層 HLOD

```
アクタ群 → HLOD0（LODLevel=0, HLODLayer=A）
                → HLOD1（LODLevel=1, HLODLayer=A.ParentLayer=B）
                         → HLOD2（LODLevel=2, ...）
```

遠距離になるほど上位 LOD レベルの HLOD が表示される。

---

## UHLODBuilder — ビルド処理

```cpp
UCLASS(Abstract, MinimalAPI)
class UHLODBuilder : public UObject
{
public:
    // ソースアクタのコンポーネントから HLOD コンポーネントを生成
    virtual TArray<UActorComponent*> Build(
        const FHLODBuildContext& InHLODBuildContext,
        const TArray<UActorComponent*>& InSourceComponents) const PURE_VIRTUAL(...);
};
```

### FHLODBuildContext

```cpp
struct FHLODBuildContext
{
    UWorld* World;                           // 対象ワールド
    TArray<UActorComponent*> SourceComponents; // 対象コンポーネント群
    UObject* AssetsOuter;                   // 生成アセットの外部オブジェクト
    FString AssetsBaseName;                 // 生成アセットのベース名
    FVector WorldPosition;                  // HLOD アクタのワールド位置
    double MinVisibleDistance;              // 最小表示距離
};
```

---

## HLOD ビルドフロー（エディタ）

```mermaid
sequenceDiagram
    participant Editor as UE Editor
    participant WP as UWorldPartition
    participant Hash as UWorldPartitionRuntimeHashSet
    participant Cell as UWorldPartitionRuntimeCell
    participant HLOD as AWorldPartitionHLOD
    participant Builder as UHLODBuilder

    Editor->>WP: Build HLOD（メニュー操作）
    WP->>Hash: GenerateHLOD()
    Hash->>Cell: 各セルのアクタを収集
    Cell->>HLOD: AWorldPartitionHLOD 生成（LODLevel=0）
    HLOD->>Builder: Build(FHLODBuildContext)
    Builder->>Builder: メッシュ結合 / 簡略化 / ISM 生成
    Builder-->>HLOD: UActorComponent（HLOD メッシュ）を返す
    HLOD-->>Hash: HLOD アクタをパッケージに保存
    Note over Hash: 上位 HLOD が必要な場合は LODLevel+1 で繰り返し
```

---

## アクタ側の設定

```cpp
// BP でも設定可能なアクタプロパティ
UPROPERTY(EditAnywhere, Category = "HLOD")
bool bActorIsHLODRelevant = true;   // HLOD ビルドに含めるか

UPROPERTY(EditAnywhere, Category = "HLOD")
TObjectPtr<UHLODLayer> HLODLayer;   // 使用する HLOD レイヤーアセット
```

`bActorIsHLODRelevant = false` のアクタは HLOD に含まれない（透明なボリューム・トリガー等に使用）。
