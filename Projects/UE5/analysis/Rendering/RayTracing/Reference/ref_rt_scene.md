# リファレンス：RayTracingScene.h / RayTracingScene.cpp

- グループ: a - TLAS / シーン構築
- 上位: [[a_rt_scene]]
- 関連: [[ref_rt_sbt]] | [[ref_rt_instances]]
- ソース: `Engine/Source/Runtime/Renderer/Private/RayTracing/RayTracingScene.h/.cpp`

## 概要

フレームごとの **Top Level Acceleration Structure（TLAS）** を管理するクラス群。  
`FRayTracingScene` がインスタンスリスト・RHI TLAS・フィードバックバッファを一元管理する。

---

## 主要クラス・列挙型

### `ERayTracingSceneLayer`

```cpp
enum class ERayTracingSceneLayer : uint8
{
    Base = 0,   // 通常オブジェクト（不透明・半透明）
    Decals,     // デカールジオメトリ
    FarField,   // 遠景オブジェクト
    NUM
};
```

### `FRayTracingScene`

RT シーン全体を管理するクラス。`FScene` が保持する。

#### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Layers` | `TArray<FLayer>` | 各レイヤー（Base/Decals/FarField）のデータ |
| `GeometriesToBuild` | `TArray<const FRayTracingGeometry*>` | このフレームで強制ビルドが必要なジオメトリ |
| `bUsesLightingChannels` | `bool` | ライティングチャンネル有効フラグ |
| `InitTask` | `UE::Tasks::FTask` | `BuildInitializationData()` の非同期タスク |
| `bCachedInstancesLocked` | `bool` | キャッシュドインスタンスの追加・解放を禁止するフラグ |
| `bTracingFeedbackEnabled` | `bool` | トレーシングフィードバック有効フラグ |

#### `FLayer` 内部構造

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Instances` | `TArray<FRayTracingGeometryInstance>` | レイヤー内のインスタンスリスト（[Cached \| Transient]） |
| `NumCachedInstances` | `uint32` | キャッシュドインスタンスの数 |
| `CachedInstancesFreeList` | `TArray<int32>` | 再利用可能なキャッシュドスロット |
| `Views` | `TArray<FLayerView>` | ビューごとの TLAS データ |

#### `FLayerView` 内部構造

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `RayTracingSceneRHI` | `FRayTracingSceneRHIRef` | RHI TLAS オブジェクト |
| `InstanceBuffer` | `FRDGBufferRef` | GPU インスタンスバッファ |
| `BuildScratchBuffer` | `FRDGBufferRef` | TLAS ビルド用スクラッチバッファ |
| `VisibleInstances` | `TBitArray<>` | 可視インスタンスのビットマスク |

---

## 主要関数

### `AddCachedInstance`

```cpp
FInstanceHandle AddCachedInstance(
    FRayTracingGeometryInstance Instance,
    ERayTracingSceneLayer Layer,
    const FPrimitiveSceneProxy* Proxy = nullptr,
    bool bDynamic = false,
    int32 GeometryHandle = INDEX_NONE);
```

| 引数 | 型 | 説明 |
|------|-----|------|
| `Instance` | `FRayTracingGeometryInstance` | BLAS 参照・トランスフォーム等 |
| `Layer` | `ERayTracingSceneLayer` | 追加先レイヤー |
| `bDynamic` | `bool` | 動的ジオメトリ（毎フレーム BLAS 更新）かどうか |

戻り値: `FInstanceHandle` — インスタンスを一意に識別するハンドル

### `Update`

```cpp
void Update(
    FRDGBuilder& GraphBuilder,
    FSceneUniformBuffer& SceneUniformBuffer,
    const FGPUScene* GPUScene,
    ERDGPassFlags ComputePassFlags);
```

GPU インスタンスバッファを確保し、インスタンスビルドを並列タスクとして開始する。

### `Build`

```cpp
void Build(
    FRDGBuilder& GraphBuilder,
    ERDGPassFlags ComputePassFlags,
    FRDGBufferRef DynamicGeometryScratchBuffer);
```

TLAS 構築コマンドを RDG パスとして追加する。

### `BuildInitializationData`

```cpp
void BuildInitializationData(
    bool bUseLightingChannels,
    bool bForceOpaque,
    bool bDisableTriangleCull);
```

`InitTask` として非同期で実行される。TLAS 作成前に必要な初期化データを準備する。

### `FInstanceHandle` / `FViewHandle`

```cpp
struct FInstanceHandle
{
    ERayTracingSceneLayer Layer; // レイヤー種別
    uint32 Index;               // レイヤー内インデックス

    uint32 AsUint32() const;    // Pack: Index | (Layer << 24)
    bool IsValid() const;
};
```

---

## RayTracing 名前空間（RayTracing.h）

### `FSceneOptions`

```cpp
namespace RayTracing
{
    struct FSceneOptions
    {
        bool bTranslucentGeometry = true;       // 半透明を TLAS に含める
        bool bIncludeSky = true;                // スカイを含める
        bool bLightingChannelsUsingAHS = true;  // AHS でライティングチャンネル処理

        FSceneOptions(
            const FScene& Scene,
            const FViewFamilyInfo& ViewFamily,
            const FViewInfo& View,
            EDiffuseIndirectMethod DiffuseIndirectMethod,
            EReflectionsMethod ReflectionsMethod);
    };
}
```

### 主要関数一覧

| 関数 | 説明 |
|------|------|
| `OnRenderBegin()` | フレーム開始時の状態リセット |
| `CreateGatherInstancesTaskData()` | 収集タスクデータ確保 |
| `AddView()` | ビューとオプションを収集タスクに追加 |
| `BeginGatherInstances()` | 並列インスタンス収集タスク開始 |
| `BeginGatherDynamicRayTracingInstances()` | 動的 BLAS 更新タスク開始 |
| `FinishGatherInstances()` | RDG パス発行・TLAS Build |
| `FinishGatherVisibleShaderBindings()` | 可視インスタンスのシェーダーバインド確定 |
| `GetVisibleShaderBindings()` | 確定済みシェーダーバインドデータを返す |
| `ShouldExcludeDecals()` | デカールを TLAS から除外するか判定 |
