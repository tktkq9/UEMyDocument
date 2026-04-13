# リファレンス：RayTracingInstanceCulling.h / RayTracingInstanceMask.h / RayTracingMaterialHitShaders.h

- グループ: a / d - インスタンス・マスク・マテリアルバインド
- 上位: [[a_rt_scene]] | [[d_rt_materials_sbt]]
- 関連: [[ref_rt_scene]] | [[ref_rt_sbt]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/RayTracing/RayTracingInstanceCulling.h/.cpp`
  - `Engine/Source/Runtime/Renderer/Private/RayTracing/RayTracingInstanceMask.h/.cpp`
  - `Engine/Source/Runtime/Renderer/Private/RayTracing/RayTracingMaterialHitShaders.h/.cpp`

## 概要

RT インスタンスの **フラスタムカリング**・**マスク設定**・**マテリアルヒットシェーダーバインド** の3つのサブシステム。

---

## RayTracingInstanceCulling.h

### `RayTracing::GetCullingMode`

```cpp
namespace RayTracing
{
    ECullingMode GetCullingMode(const FEngineShowFlags& ShowFlags);
}
```

ShowFlags を参照してカリングモードを返す。

### `RayTracing::ShouldConsiderCulling`

```cpp
bool ShouldConsiderCulling(
    const FRayTracingCullingParameters& CullingParameters,
    const FBoxSphereBounds& ObjectBounds,
    float MinDrawDistance);
```

| 引数 | 説明 |
|------|------|
| `CullingParameters` | カリング設定（半径・角度・フラグ等） |
| `ObjectBounds` | プリミティブの境界ボックス+球 |
| `MinDrawDistance` | 最小描画距離 |

設定値のチェックは行わない。事前に設定値チェックが済んでいることを前提とする。

### `RayTracing::CullPrimitiveByFlags`

```cpp
bool CullPrimitiveByFlags(
    const FRayTracingCullingParameters& CullingParameters,
    const FScene* RESTRICT Scene,
    int32 PrimitiveIndex);
```

RT フラグ（`bVisibleInRayTracing`, `bHidden` 等）によるカリング。  
境界ボックスは考慮せず、フラグのみで判定する。

### `RayTracing::ShouldCullBounds`

```cpp
bool ShouldCullBounds(
    const FRayTracingCullingParameters& CullingParameters,
    const FBoxSphereBounds& ObjectBounds,
    float MinDrawDistance,
    bool bIsFarFieldPrimitive);
```

設定値チェック込みの完全カリング判定。内部で `ShouldConsiderCulling()` を使用。

### `GetRayTracingCullingRadius`

```cpp
float GetRayTracingCullingRadius();
```

`r.RayTracing.Culling.Radius` CVar の値を返す。

---

## RayTracingInstanceMask.h

### `FRayTracingMaskAndFlags`

```cpp
struct FRayTracingMaskAndFlags
{
    uint8 Mask = 0xFF;              // インスタンスマスク（エフェクト別の除外に使用）
    uint8 bForceOpaque : 1;        // Any-Hit Shader を無効化
    uint8 bDoubleSided : 1;        // 両面ヒット登録
    uint8 bReverseCulling : 1;     // 表裏反転
    uint8 bAnySegmentsDecal : 1;   // セグメントの一部がデカール
    uint8 bAllSegmentsDecal : 1;   // 全セグメントがデカール
    uint8 bAllSegmentsTranslucent : 1; // 全セグメントが半透明
};
```

### `ERayTracingType`

```cpp
enum class ERayTracingType : uint8
{
    RayTracing,       // 通常レイトレーシング
    PathTracing,      // パストレーシング
    LightMapTracing,  // ライトマップ焼き込み
};
```

マッシュコマンドに保存され、Invalidation の判断に使用される。

### `BlendModeToRayTracingInstanceMask`

```cpp
RENDERER_API uint8 BlendModeToRayTracingInstanceMask(
    const EBlendMode BlendMode,
    bool bIsDitherMasked,
    bool bCastShadow,
    ERayTracingType RayTracingType);
```

ブレンドモードからインスタンスマスクのビットパターンを生成する。

### `BuildRayTracingInstanceMaskAndFlags`

```cpp
FRayTracingMaskAndFlags BuildRayTracingInstanceMaskAndFlags(
    const FRayTracingInstance& Instance,
    const FPrimitiveSceneProxy& PrimitiveSceneProxy);
```

`FRayTracingInstance` とプロキシ情報からマスク＋フラグを生成（インスタンスは変更しない）。

### `SetupRayTracingMeshCommandMaskAndStatus`

```cpp
void SetupRayTracingMeshCommandMaskAndStatus(
    FRayTracingMeshCommand& MeshCommand,
    const FMeshBatch& MeshBatch,
    const FPrimitiveSceneProxy* PrimitiveSceneProxy,
    const FMaterial& MaterialResource,
    ERayTracingType RayTracingType);
```

`FRayTracingMeshCommand` にマスクとステータスを設定する。  
`FRayTracingMeshProcessor::BuildRayTracingMeshCommands()` 内から呼ばれる。

---

## RayTracingMaterialHitShaders.h

### `FRayTracingMeshProcessor`

```cpp
class FRayTracingMeshProcessor
{
public:
    FRayTracingMeshProcessor(
        FRayTracingMeshCommandContext* InCommandContext,
        const FScene* InScene,
        const FSceneView* InViewIfDynamicMeshCommand,
        ERayTracingType InRayTracingType);

    // マテリアルを解析してヒットグループシェーダーバインドを生成
    void AddMeshBatch(
        const FMeshBatch& MeshBatch,
        uint64 BatchElementMask,
        const FPrimitiveSceneProxy* PrimitiveSceneProxy);

private:
    FRayTracingMeshCommandContext* CommandContext; // SBT レコード書き込み先
    const FScene* Scene;
    ERayTracingType RayTracingType;
};
```

#### `bCanBeCached` の判定

```cpp
// グローバル UBO なしの場合のみキャッシュ可能
RayTracingMeshCommand.bCanBeCached = !RayTracingMeshCommand.HasGlobalUniformBufferBindings();
```

### デフォルトシェーダー取得関数

| 関数 | 返すシェーダー | 用途 |
|------|-------------|------|
| `GetRayTracingDefaultMissShader()` | デフォルト Miss | レイが何にも当たらなかった場合 |
| `GetRayTracingDefaultOpaqueShader()` | デフォルト不透明 HitGroup | マテリアルなしジオメトリ |
| `GetRayTracingDefaultHiddenShader()` | HiddenMaterialHitGroup | 不可視インスタンス用 |

### グローバルシェーダー

| クラス | ペイロード | 説明 |
|--------|-----------|------|
| `FHiddenMaterialHitGroup` | `RayTracingMaterial` | 即 miss するダミーヒットグループ |
| `FOpaqueShadowHitGroup` | `RayTracingMaterial` | 不透明シャドウ用 any-hit 無効化ヒットグループ |
| `FDefaultCallableShader` | `Decals` | デカール用デフォルト callable シェーダー |

### `SetRayTracingShaderBindings`

```cpp
void SetRayTracingShaderBindings(
    FRHICommandList& RHICmdList,
    FSceneRenderingBulkObjectAllocator& Allocator,
    FViewInfo::FRayTracingData& RayTracingData);
```

`FRayTracingLocalShaderBindingWriter` が書き込んだデータを  
RHI コマンドリストに発行する最終ステップ。

---

## 使用箇所

- [[ref_rt_scene]] — `BeginGatherInstances()` 内でカリング・マスク設定を呼び出す
- [[ref_rt_sbt]] — `FRayTracingMeshProcessor` が `AddMeshBatch()` でシェーダーを SBT に書き込む
