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

## コード実行フロー

### テッセレーション フロー

```
r.Nanite.Tessellation = 1 の場合のみ有効

IRenderer::DrawGeometry()
  │  通常のクラスターカリングに加えて:
  │
  ├─ FRasterResults::VisiblePatches バッファへ
  │    可視クラスターをパッチとして出力
  │
  └─ [テッセレーションパス]
       TessellationTable のパターン（ルックアップテーブル）を参照
         → パッチを細分化（DicingRate に基づくマイクロポリゴン生成）
         → SW ラスタライザ（Compute）で VisBuffer に書き込み
```

### ボクセル描画 フロー

```
デバッグ / 内部ビジュアライゼーション用

Nanite::DrawVisibleBricks()   Voxel.h
  → シーンをボクセルグリッドに区切る
  → 可視ブリックを Compute または Draw で描画
  → デバッグビュー上に表示
```

### テッセレーションテーブル 生成フロー

```
ビルド時（NANITE_BUILD_TESSELLATION_TABLE マクロ有効時）:
  TessellationTable.cpp でテッセレーションパターンを事前計算
  → バイナリファイルに出力

ランタイム（通常）:
  事前生成済みバイナリをロードしてシェーダーバッファにアップロード
  → DrawGeometry() 内のテッセレーションパスが参照
```

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 説明 |
|--------------|---------|------|
| `FRasterResults::VisiblePatches` | `NaniteCullRaster.h` | テッセレーション用可視パッチリスト |
| `Nanite::DrawVisibleBricks()` | `Voxel.h` | ボクセルブリック描画（デバッグ用）|
| `TessellationTable` | `TessellationTable.cpp` | テッセレーションパターン事前生成・ロード |

---

## 関連リファレンス

| リファレンス | 対象ソース | 主な内容 |
|------------|---------|---------|
| [[ref_nanite_tessellation_table]] | `TessellationTable.cpp` | テッセレーションテーブルの生成フロー・定数 |
| [[ref_nanite_voxel]] | `Voxel.h/.cpp` | DrawVisibleBricks 関数・ボクセルブリック構造 |
