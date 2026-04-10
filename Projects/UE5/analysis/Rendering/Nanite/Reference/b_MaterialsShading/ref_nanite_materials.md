# リファレンス：NaniteMaterials.h / NaniteMaterials.cpp

- グループ: b - Materials & Shading
- 上位: [[b_nanite_materials_shading]]
- 関連: [[ref_nanite_materials_scene_ext]] | [[ref_nanite_shading]] | [[ref_nanite_draw_list]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteMaterials.h/.cpp`

---

## 概要

Nanite のマテリアルスロット・ビン分類・デバッグビューメタデータを定義するファイル。  
各プリミティブが「どのラスタライズビン」「どのシェーディングビン」に属するかを管理する。

---

## 主要クラス・構造体

```cpp
namespace Nanite
{

// マテリアルスロット（プリミティブごとに割り当てられるビン索引）
struct FNaniteMaterialSlot
{
    uint32 RasterBin  : 12;  // ラスタライズビン ID（0〜4095）
    uint32 ShadingBin : 12;  // シェーディングビン ID（0〜4095）
    uint32 Flags      : 8;   // 予約フラグ
};

// シェーディングビン（VisBuffer のマテリアル深度から参照）
struct FNaniteShadingBin
{
    uint32 DataByteOffset;   // ビンデータ（FNaniteShadingPipeline 等）へのオフセット
    uint32 MaterialFlags;    // シェーダー選択・機能フラグ
};

// マテリアルデバッグビュー情報（r.Nanite.Visualize=MaterialID 等で使用）
struct FNaniteMaterialDebugViewInfo
{
    FLinearColor Color;      // マテリアルごとの識別色
    FString MaterialName;    // マテリアル名
};

// マテリアル設定パラメータ（シェーダーへのバインド用）
struct FNaniteMaterialsParameters
{
    // GPU バッファ SRV（シェーダーからマテリアルスロットを参照するため）
    FRDGBufferSRVRef MaterialSlotBuffer;
    FRDGBufferSRVRef ShadingBinBuffer;
};

} // namespace Nanite

// マテリアルビットフラグのパック
uint32 PackMaterialBitFlags(
    const FMaterial& Material,
    bool bReverseWindingOrder,
    bool bHasProgrammableRaster);
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Nanite.ProgrammableRaster` | 1 | マスク・PDO・WPO 等プログラマブルラスタ有効 |
| `r.Nanite.MaterialCache` | 1 | マテリアルパイプラインキャッシュ有効 |
