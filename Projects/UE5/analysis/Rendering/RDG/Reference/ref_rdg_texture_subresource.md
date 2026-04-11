# リファレンス：RenderGraphTextureSubresource.h

- グループ: b - Resources
- 上位: [[b_rdg_resources]]
- 関連: [[ref_rdg_resources]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphTextureSubresource.h`

## 概要

テクスチャのサブリソース（Mip / ArraySlice / PlaneSlice）を表すデータ構造群。  
RDG 内部でバリア範囲の計算や SRV/UAV の部分ビュー生成に使用される。

---

## FRDGTextureSubresource

```cpp
struct FRDGTextureSubresource
{
    uint32 MipIndex   : 8;   // Mip レベルインデックス（0 = フル解像度）
    uint32 PlaneSlice : 8;   // プレーンスライス（深度・ステンシルの分離に使用）
    uint32 ArraySlice : 16;  // テクスチャ配列スライス

    // デフォルト: Mip 0, PlaneSlice 0, ArraySlice 0
    FRDGTextureSubresource();

    // 指定スライスで構築
    FRDGTextureSubresource(uint32 InMipIndex, uint32 InArraySlice, uint32 InPlaneSlice);

    // 比較演算子（==, !=, <, <=, >, >=）
};
```

---

## FRDGTextureSubresourceLayout

テクスチャ全体のサブリソース数を表す構造体。

```cpp
struct FRDGTextureSubresourceLayout
{
    uint32 NumMips;         // Mip レベル数
    uint32 NumPlaneSlices;  // プレーンスライス数（通常 1、深度/ステンシルは 2）
    uint32 NumArraySlices;  // 配列スライス数（Cube は 6）

    FRDGTextureSubresourceLayout();
    FRDGTextureSubresourceLayout(uint32 InNumMips, uint32 InNumArraySlices, uint32 InNumPlaneSlices);

    // このサブリソース数に対する線形インデックスを計算
    uint32 GetSubresourceIndex(const FRDGTextureSubresource& Subresource) const;

    // 総サブリソース数 = NumMips × NumArraySlices × NumPlaneSlices
    uint32 GetSubresourceCount() const;
};
```

---

## TRDGTextureSubresourceArray

```cpp
// サブリソースごとのデータ配列（FRDGTextureSubresourceLayout と組み合わせて使用）
template <typename InElementType, typename InAllocatorType = FDefaultAllocator>
using TRDGTextureSubresourceArray = TArray<InElementType, InAllocatorType>;
```

### 使用例

```cpp
// テクスチャの全サブリソースに ERHIAccess を割り当てる
TRDGTextureSubresourceArray<ERHIAccess> AccessArray;
AccessArray.SetNum(Layout.GetSubresourceCount());
AccessArray[Layout.GetSubresourceIndex({Mip, ArraySlice, Plane})] = ERHIAccess::SRVGraphics;
```
