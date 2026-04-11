# リファレンス：RenderGraphTextureSubresource.h

- グループ: b - Resources
- 上位: [[b_rdg_resources]]
- 関連: [[ref_rdg_resources]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphTextureSubresource.h`

## 概要

テクスチャのサブリソース（Mip / ArraySlice / PlaneSlice）を識別・操作するための  
データ構造群。RDG 内部でバリア範囲の計算と SRV/UAV の部分ビュー生成に使用される。

---

## FRDGTextureSubresource

1 つのサブリソース（特定の Mip × ArraySlice × PlaneSlice の組み合わせ）を表す。

```cpp
struct FRDGTextureSubresource
{
    uint32 MipIndex   : 8;   // Mip レベル（0 = フル解像度）
    uint32 PlaneSlice : 8;   // プレーンスライス（深度/ステンシル分離に使用）
    uint32 ArraySlice : 16;  // テクスチャ配列インデックス

    FRDGTextureSubresource();  // デフォルト: {0, 0, 0}
    FRDGTextureSubresource(uint32 InMipIndex, uint32 InArraySlice, uint32 InPlaneSlice);

    // 比較演算子: ==, !=, <, <=, >, >=
};
```

### メンバ変数

| 変数 | ビット幅 | 説明 |
|-----|---------|------|
| `MipIndex` | 8 bit | ミップレベル（最大 255） |
| `PlaneSlice` | 8 bit | プレーンスライス（通常 0 / 深度=0, ステンシル=1） |
| `ArraySlice` | 16 bit | 配列スライス（Cube=0〜5、配列テクスチャ） |

### 使用箇所
- [[ref_rdg_resources]] `FRDGTexture::GetSubresource(uint32 Index)` の戻り値
- RDG 内部のバリア計算（`FRDGSubresourceState` の配列インデックス）

---

## FRDGTextureSubresourceLayout

テクスチャ全体のサブリソース数と次元構造を表す。  
`FRDGTextureSubresource` とペアでインデックス計算に使う。

```cpp
struct FRDGTextureSubresourceLayout
{
    uint32 NumMips        : 8;   // ミップレベル数
    uint32 NumPlaneSlices : 8;   // プレーンスライス数（通常 1、深度/ステンシルは 2）
    uint32 NumArraySlices : 16;  // 配列スライス数（Cube は 6）

    // コンストラクタ
    FRDGTextureSubresourceLayout();
    FRDGTextureSubresourceLayout(uint32 InNumMips, uint32 InNumArraySlices, uint32 InNumPlaneSlices);
    FRDGTextureSubresourceLayout(const FRHITextureDesc& Desc);  // RHI 記述子から構築

    // 総サブリソース数 = NumMips × NumArraySlices × NumPlaneSlices
    uint32 GetSubresourceCount() const;

    // サブリソース → 線形インデックス（バリア状態配列のアクセスに使用）
    uint32 GetSubresourceIndex(FRDGTextureSubresource Subresource) const;

    // 線形インデックス → サブリソース
    FRDGTextureSubresource GetSubresource(uint32 Index) const;

    // 比較演算子: ==, !=
};
```

### インデックス計算式

```cpp
// GetSubresourceIndex の実装
Index = MipIndex + (ArraySlice * NumMips) + (PlaneSlice * NumMips * NumArraySlices);
```

### 使用箇所
- [[ref_rdg_resources]] `FRDGTexture` の内部メンバ `Layout`
- `FRDGTexture::GetSubresourceLayout()` の戻り値

---

## FRDGTextureSubresourceRange

テクスチャの特定範囲のサブリソースを表す。  
SRV/UAV の部分ビューや、バリアの一部適用に使用。

```cpp
struct FRDGTextureSubresourceRange
{
    uint32 MipIndex   : 8;
    uint32 PlaneSlice : 8;
    uint32 ArraySlice : 16;
    uint32 NumMips    : 8;
    uint32 NumPlaneSlices : 8;
    uint32 NumArraySlices : 16;

    // FRDGTextureSubresourceLayout から全範囲で構築
    explicit FRDGTextureSubresourceRange(FRDGTextureSubresourceLayout Layout);

    // 範囲内のサブリソース数
    uint32 GetSubresourceCount() const;

    // 最小・最大サブリソースの取得
    FRDGTextureSubresource GetMinSubresource() const;
    FRDGTextureSubresource GetMaxSubresource() const;

    // 全サブリソースを列挙するイテレータ
    template <typename TFunction>
    void EnumerateSubresources(TFunction Function) const;

    // テクスチャ全体を指すかどうか
    bool IsWholeResource(const FRDGTextureSubresourceLayout& Layout) const;

    // 比較演算子: ==, !=
};
```

### 使用箇所
- [[ref_rdg_resources]] `FRDGTexture::GetSubresourceRange()` の戻り値（全体範囲）
- `FRDGTexture::GetSubresourceRangeSRV()` の戻り値（SRV 対象範囲）
- RDG 内部バリア計算でバリア適用範囲を指定するため

---

## TRDGTextureSubresourceArray

サブリソースごとのデータを格納する配列型エイリアス。  
RDG 内部では `FRDGSubresourceState` の配列として使用される。

```cpp
template <typename InElementType, typename InAllocatorType = FDefaultAllocator>
using TRDGTextureSubresourceArray = TArray<InElementType, InAllocatorType>;

// 使用例: バリア状態配列
using FRDGTextureSubresourceState = TRDGTextureSubresourceArray<FRDGSubresourceState*, FRDGArrayAllocator>;
```

### 使用例

```cpp
// テクスチャの全サブリソースに ERHIAccess を割り当てる
FRDGTextureSubresourceLayout Layout = Texture->GetSubresourceLayout();
TRDGTextureSubresourceArray<ERHIAccess> AccessArray;
AccessArray.SetNum(Layout.GetSubresourceCount());

// Mip2, ArraySlice0, PlaneSlice0 のアクセスを設定
uint32 Index = Layout.GetSubresourceIndex({2, 0, 0});
AccessArray[Index] = ERHIAccess::SRVGraphics;
```
