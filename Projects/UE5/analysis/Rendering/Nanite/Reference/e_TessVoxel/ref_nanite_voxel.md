# リファレンス：Voxel.h / Voxel.cpp

- グループ: e - Tessellation & Voxel
- 上位: [[e_nanite_tess_voxel]]
- 関連: [[ref_nanite_visualize]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/Voxel.h/.cpp`

---

## 概要

Nanite シーンをボクセルブリックに変換してデバッグ表示するモジュール。  
シーン内のジオメトリを固定グリッドに区切り、各セルを1ブリックとして可視化する。  
主に開発・デバッグ用途（エンジン内部の可視化）。

---

## 主要関数

```cpp
namespace Nanite
{

// 表示対象ボクセルブリックの描画
void DrawVisibleBricks(
    FRDGBuilder& GraphBuilder,
    const FScene& Scene,
    const FViewInfo& View,
    FRDGTextureRef SceneDepth,
    FRDGTextureRef SceneColor);

} // namespace Nanite
```

---

## 主要な概念

```
ボクセルブリック:
  シーンを 3D グリッドに分割した最小単位。
  各ブリックに「ジオメトリが存在するか」のフラグを持つ。

  用途:
  - デバッグビジュアライゼーション（密度・分布の確認）
  - エンジン内部の空間分割確認
```
