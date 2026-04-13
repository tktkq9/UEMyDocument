# リファレンス：RayTracingShaderBindingTable.h / .cpp

- グループ: d - マテリアルヒットシェーダー・SBT 管理
- 上位: [[d_rt_materials_sbt]]
- 関連: [[ref_rt_scene]] | [[ref_rt_instances]]
- ソース: `Engine/Source/Runtime/Renderer/Private/RayTracing/RayTracingShaderBindingTable.h/.cpp`

## 概要

Ray Tracing の **Shader Binding Table（SBT）** を管理するクラス群。  
`FRayTracingShaderBindingTable` がヒットグループレコードの静的・動的割り当てを担い、  
`FRayTracingSBTAllocation` が個々の割り当て単位を表す。

---

## 主要クラス・列挙型

### `ERayTracingShaderBindingLayer`

```cpp
enum class ERayTracingShaderBindingLayer : uint8
{
    Base = 0,   // 通常レンダリング用
    Decals,     // デカール用
    NUM
};
```

### `FRayTracingSBTAllocation`

個々のジオメトリセグメントに割り当てられた SBT レコード範囲を表す。

#### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `BaseRecordIndex` | `uint32` | SBT 内の先頭レコードインデックス |
| `RecordsPerLayer` | `uint32` | 1レイヤーあたりのレコード数 |
| `NumRecords` | `uint32` | 全レコード数（全レイヤー合計） |
| `AllocatedLayers` | `ERayTracingShaderBindingLayerMask` | 割り当て済みレイヤーのビットマスク |
| `Geometry` | `const FRHIRayTracingGeometry*` | 割り当てに使ったジオメトリ（共有判定用） |
| `Flags` | `FRayTracingCachedMeshCommandFlags` | マテリアルフラグ（共有判定用） |

#### 主要メソッド

| メソッド | 説明 |
|---------|------|
| `GetInstanceContributionToHitGroupIndex(Layer)` | 指定レイヤーの InstanceContribution 値を返す |
| `GetRecordIndex(Layer, SegmentIndex)` | 指定レイヤー・セグメントの SBT レコードインデックスを返す |
| `HasLayer(Layer)` | 指定レイヤーが割り当て済みか |
| `IsValid()` | NumRecords > 0 かどうか |

---

### `FRayTracingShaderBindingTable`

#### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `NumShaderSlotsPerGeometrySegment` | `uint32` | 1セグメントあたりのシェーダースロット数（全体固定値） |
| `PersistentSBTs` | `TArray<FPersistentSBTData>` | 永続 SBT の実体リスト |
| `DirtyPersistentRecords` | `TBitArray<>` | 更新が必要な永続レコードのビットマスク |
| `StaticRangeAllocator` | `FSpanAllocator` | 静的レコード範囲のアロケータ |
| `TrackedAllocationMap` | `TMap<FAllocationKey, FRefCountedAllocation>` | ジオメトリ+フラグで共有管理する割り当てマップ |
| `ActiveDynamicAllocations` | `TArray<FRayTracingSBTAllocation*>` | 現在アクティブな動的割り当て |
| `NumMissShaderSlots` | `uint32` | Miss シェーダースロット数（最低 1） |
| `NumCallableShaderSlots` | `uint32` | Callable シェーダースロット数 |

#### 主要メソッド

| メソッド | 説明 |
|---------|------|
| `AllocateStaticRange(SegmentCount, Geometry, Flags)` | 静的レコード範囲を割り当て（同一 Geometry+Flags なら共有） |
| `FreeStaticRange(Allocation)` | 静的割り当てを解放（RefCount 管理） |
| `AllocateDynamicRange(AllocatedLayers, SegmentCount)` | 動的レコード範囲を割り当て（フレームごとにリセット） |
| `ResetDynamicAllocationData()` | 動的割り当てをリセット（フレーム終了時） |
| `GetDirtyBindings(VisibleBindings, bForceAllDirty)` | 更新が必要なバインドデータを返す |
| `AllocatePersistentSBTID(RHICmdList, ShaderBindingMode)` | 永続 SBT ID を確保 |
| `CheckPersistentRHI(RHICmdList, MaxLocalBindingDataSize)` | 永続 SBT の RHI オブジェクトを検証・再生成 |
| `AllocateTransientRHI(...)` | 1フレーム用の一時 SBT を確保 |
| `GetNumGeometrySegments()` | 現在割り当てられている全セグメント数 |

---

## `FRayTracingLocalShaderBindingWriter`

並列タスク内でシェーダーバインドデータを書き込むためのヘルパークラス。  
チャンク（最大 1024 バインド/チャンク）形式で内部リストを管理する。

```cpp
class FRayTracingLocalShaderBindingWriter
{
    // インラインパラメータ付きバインドを追加
    FRayTracingLocalShaderBindings& AddWithInlineParameters(
        uint32 NumUniformBuffers, uint32 LooseDataSize = 0);

    // 外部パラメータ付きバインドを追加
    FRayTracingLocalShaderBindings& AddWithExternalParameters();

    // RHI コマンドリストに全バインドを発行
    void Commit(FRHICommandList& RHICmdList,
                FRHIShaderBindingTable* SBT,
                FRayTracingPipelineState* Pipeline,
                bool bCopyDataToInlineStorage) const;
};
```

---

## `AddRayTracingLocalShaderBindingWriterTasks`

```cpp
template <typename FunctionType>
void AddRayTracingLocalShaderBindingWriterTasks(
    FRDGBuilder& GraphBuilder,
    TConstArrayView<FRayTracingShaderBindingData> DirtyPersistentRayTracingShaderBindings,
    TArray<FRayTracingLocalShaderBindingWriter*, SceneRenderingAllocator>& ShaderBindingWriters,
    FunctionType SetupBindingsFunction);
```

- 1024 バインドごとに並列 `FRDGBuilder::AddSetupTask()` を生成
- 各タスクは `SetupBindingsFunction` を呼び出しシェーダーバインドを書き込む

---

## `SetRayTracingShaderBindings`

```cpp
void SetRayTracingShaderBindings(
    FRHICommandList& RHICmdList,
    FSceneRenderingBulkObjectAllocator& Allocator,
    FViewInfo::FRayTracingData& RayTracingData);
```

並列タスクで書き込んだバインドデータを RHI コマンドリストに発行する最終ステップ。
