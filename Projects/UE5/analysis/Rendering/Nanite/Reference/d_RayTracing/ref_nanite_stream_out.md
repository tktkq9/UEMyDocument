# リファレンス：NaniteStreamOut.h / NaniteStreamOut.cpp

- グループ: d - Ray Tracing
- 上位: [[d_nanite_ray_tracing]]
- 関連: [[ref_nanite_ray_tracing]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteStreamOut.h/.cpp`

---

## 概要

Nanite のクラスター単位でページ管理されているメッシュデータを  
GPU から CPU に抽出（ストリームアウト）するシステム。  
主に BLAS 構築（`FRayTracingManager`）が必要とする頂点・インデックスデータを供給する。

---

## 主要構造体

```cpp
namespace Nanite
{

// ストリームアウト対象メッシュデータのヘッダ
struct FStreamOutMeshDataHeader
{
    uint32 NumVertices;       // 頂点数
    uint32 NumTriangles;      // 三角形数
    uint32 NumSegments;       // セグメント数（マテリアル区切り）
    uint32 VertexBufferOffset;
    uint32 IndexBufferOffset;
};

// メッシュデータのセグメント（マテリアル別区間）
struct FStreamOutMeshDataSegment
{
    uint32 FirstIndex;        // このセグメントの先頭インデックス
    uint32 NumTriangles;      // セグメント内の三角形数
    uint32 MaterialIndex;     // マテリアルインデックス
};

// ストリームアウトリクエスト
struct FStreamOutRequest
{
    FPrimitiveSceneInfo* Primitive;
    uint32 LODIndex;
    FStreamOutMeshDataHeader Header;
    TArray<FStreamOutMeshDataSegment> Segments;
    // 完了コールバック
    TFunction<void(TArrayView<const uint8> VertexData,
                   TArrayView<const uint32> IndexData)> OnComplete;
};

} // namespace Nanite

// ストリームアウト実行（GPU ジョブ発行）
void StreamOutData(
    FRDGBuilder& GraphBuilder,
    const FScene& Scene,
    TArrayView<FStreamOutRequest> Requests);
```

---

## 主要関数

| 関数 | 説明 |
|------|------|
| `StreamOutData(FRDGBuilder&, FScene&, Requests)` | リクエスト一覧を GPU で処理し頂点・インデックスデータを CPU に戻す |
