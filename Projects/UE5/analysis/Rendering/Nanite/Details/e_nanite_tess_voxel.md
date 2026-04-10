# Nanite Tessellation & Voxel（テッセレーション・ボクセル）

- 上位: [[03_nanite_overview]]
- 関連: [[a_nanite_cull_raster]]

---

## 概要

Nanite の2つの拡張表現：

1. **マイクロポリゴンテッセレーション**（実験的機能）  
   ディスプレイスメントマップを用いてポリゴンをピクセル以下のサイズに細分化し、  
   Compute ラスタライザで処理する。TessellationTable がテッセレーションパターンを管理する。

2. **ボクセル表現**（デバッグ・内部用）  
   シーンをボクセルブリックに変換して描画する機能。  
   主に開発・デバッグ用途。

---

## テッセレーションの仕組み

```
r.Nanite.Tessellation = 1 で有効化（デフォルト無効）

テッセレーションフロー:
  1. クラスターをパッチとして出力（NaniteCullRaster → FRasterResults::VisiblePatches）
  2. TessellationTable のパターンに従ってパッチを細分化
  3. Compute ラスタライザ（SW Raster）で微小三角形を VisBuffer に書き込む

主要 CVar:
  r.Nanite.Tessellation       = 0    有効/無効
  r.Nanite.DicingRate         = 1.0  マイクロポリゴンの目標サイズ（px）
  r.Nanite.MaxPatchesPerGroup = 1    パッチバッチの上限
```

## TessellationTable の構造

```cpp
// テッセレーションパターンをビルド時に事前生成（またはランタイム読み込み）
// NANITE_BUILD_TESSELLATION_TABLE マクロで切り替え

// テッセレーションパターン（各パッチの三角形分割方式をルックアップテーブルで定義）
// 出力: 各パッチに対応する頂点インデックスリスト
```

---

## ボクセルの仕組み

```cpp
// Voxel.h
namespace Nanite
{
    // 可視ボクセルブリックの描画
    void DrawVisibleBricks(
        FRDGBuilder& GraphBuilder,
        const FScene& Scene,
        const FViewInfo& View,
        ...
    );
}

// ボクセルブリック: シーンを固定グリッドに区切り、各セルを1ブリックとして管理
// 主にシーン表現のデバッグ・内部ビジュアライゼーションに使用
```

---

## 関連ソースファイル

| ファイル | 役割 |
|---------|------|
| `TessellationTable.cpp` | テッセレーションパターンテーブルのビルド/読み込み |
| `Voxel.h/.cpp` | ボクセルブリックの描画関数 |

---

## 関連リファレンス

| リファレンス | 対象ソース | 主な内容 |
|------------|---------|---------|
| [[ref_nanite_tessellation_table]] | `TessellationTable.cpp` | テッセレーションテーブルの生成フロー・定数 |
| [[ref_nanite_voxel]] | `Voxel.h/.cpp` | DrawVisibleBricks 関数・ボクセルブリック構造 |
