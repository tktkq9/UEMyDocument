# リファレンス：NaniteRayTracing.h / NaniteRayTracing.cpp

- グループ: d - Ray Tracing
- 上位: [[d_nanite_ray_tracing]]
- 関連: [[ref_nanite_stream_out]] | [[ref_nanite_shared]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteRayTracing.h/.cpp`

---

## 概要

Nanite のクラスター構造メッシュから HW レイトレーシング用の  
加速構造体（BLAS: Bottom Level Acceleration Structure）を構築・管理するシステム。  
ストリーミング状態に応じて BLAS を増分構築し、Lumen HW RT・DXR シャドウ等に提供する。

---

## 主要クラス

```cpp
namespace Nanite
{

// レイトレーシング加速構造体管理
class FRayTracingManager
{
public:
    // プリミティブの登録・削除
    void Add(FPrimitiveSceneInfo* SceneInfo);
    void Remove(FPrimitiveSceneInfo* SceneInfo);

    // ストリーミング更新（毎フレーム: ページロード状況に応じて BLAS 更新キューを操作）
    void UpdateStreaming(FRDGBuilder& GraphBuilder);

    // GPU ジョブ発行: キューに積まれた BLAS を実際に構築
    void ProcessUpdateRequests(FRDGBuilder& GraphBuilder);
    void ProcessBuildRequests(FRDGBuilder& GraphBuilder);

    // 完成した BLAS の参照取得
    FRayTracingGeometry* GetRayTracingGeometry(
        const FPrimitiveSceneInfo* SceneInfo) const;

    // 有効かどうか（HW RT がサポートされているか）
    bool IsSupported() const;

private:
    // 内部ジオメトリデータ（クラスターメッシュ情報）
    struct FInternalData
    {
        FRayTracingGeometry Geometry;    // BLAS
        uint32 LODIndex;
        bool bDynamic;
        bool bBuilt;
    };

    TMap<FPrimitiveSceneInfo*, FInternalData> GeometryMap;

    // BLAS 構築待機キュー
    struct FPendingBuild
    {
        FPrimitiveSceneInfo* SceneInfo;
        uint32 LODIndex;
        bool bDynamic;
    };
    TArray<FPendingBuild> PendingBuilds;
    TArray<FPendingBuild> PendingUpdates;
};

} // namespace Nanite
```

---

## 主要関数

| 関数 | 説明 |
|------|------|
| `Add(FPrimitiveSceneInfo*)` | プリミティブを管理対象として登録 |
| `Remove(FPrimitiveSceneInfo*)` | プリミティブを削除し BLAS を解放 |
| `UpdateStreaming(FRDGBuilder&)` | ページストリーミング状態に応じてキューを更新 |
| `ProcessBuildRequests(FRDGBuilder&)` | 待機キューの BLAS を GPU で構築 |
| `GetRayTracingGeometry(SceneInfo)` | 完成した BLAS を返す（未完の場合は null） |
