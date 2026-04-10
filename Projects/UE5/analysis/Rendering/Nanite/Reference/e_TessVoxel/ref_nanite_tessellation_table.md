# リファレンス：TessellationTable.cpp

- グループ: e - Tessellation & Voxel
- 上位: [[e_nanite_tess_voxel]]
- 関連: [[ref_nanite_cull_raster]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/TessellationTable.cpp`

---

## 概要

Nanite のマイクロポリゴンテッセレーション（実験的機能）で使用する  
テッセレーションパターンテーブルを管理するモジュール。  
`NANITE_BUILD_TESSELLATION_TABLE` マクロが有効な場合はビルド時に生成し、  
無効な場合は事前生成されたバイナリから読み込む。

---

## 概要構造

```cpp
// テッセレーションパターン数・パラメータ（定数）
// Nanite SW ラスタライザが参照する分割パターンのルックアップテーブル
// 各エントリ: パッチを N×M グリッドに分割した場合の頂点インデックス列

namespace Nanite
{

// テッセレーションテーブルビルド（ツール用）
// NANITE_BUILD_TESSELLATION_TABLE を定義して有効化
#if NANITE_BUILD_TESSELLATION_TABLE
void BuildTessellationTable(const FString& OutputPath);
#endif

// テッセレーションテーブル読み込み（ランタイム用）
bool LoadTessellationTable(TArray<uint8>& OutData);

} // namespace Nanite
```

---

## テッセレーション有効化 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Nanite.Tessellation` | 0 | マイクロポリゴンテッセレーション有効（実験的） |
| `r.Nanite.DicingRate` | 1.0 | マイクロポリゴンの目標サイズ（px） |
| `r.Nanite.MaxPatchesPerGroup` | 1 | パッチバッチ上限 |
