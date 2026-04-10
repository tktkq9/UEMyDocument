# リファレンス：NaniteComposition.h / NaniteComposition.cpp

- グループ: a - CullRaster
- 上位: [[a_nanite_cull_raster]]
- 関連: [[ref_nanite_cull_raster]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteComposition.h/.cpp`

---

## 概要

Nanite ラスタライズで生成した VisBuffer64 を、  
エンジンが期待する SceneDepth / CustomDepth / Stencil フォーマットに変換・書き出すモジュール。  
H-Tile の再要約や深度デコンプレスも担当する。

---

## 主要構造体

```cpp
namespace Nanite
{

// カスタム深度処理コンテキスト
struct FCustomDepthContext
{
    FRDGTextureRef DepthTarget;     // カスタム深度テクスチャ
    FRDGTextureRef StencilTarget;   // カスタムステンシルテクスチャ
    bool bHasAnyPrimitives;         // 処理対象プリミティブがあるか
};

} // namespace Nanite
```

---

## 主要関数

| 関数 | 説明 |
|------|------|
| `EmitDepthTargets(...)` | VisBuffer64 → SceneDepth / SceneStencil に変換・書き出し |
| `InitCustomDepthStencilContext(...)` | カスタム深度処理の初期化（対象プリミティブ収集） |
| `EmitCustomDepthStencilTargets(...)` | CustomDepth / CustomStencil への書き出し |
| `FinalizeCustomDepthStencil(...)` | カスタム深度の確定・H-Tile 更新 |
| `MarkSceneStencilRects(...)` | ステンシルの矩形マーク |
| `EmitSceneDepthRects(...)` | 矩形単位の深度書き出し（部分更新用） |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Nanite.ResummarizeHTile` | 1 | 深度出力後に H-Tile を再要約するか |
| `r.Nanite.DecompressDepth` | 0 | 深度バッファを強制デコンプレス |
| `r.Nanite.CustomDepthExportMethod` | 1 | カスタム深度の出力方式（0=Copy, 1=Shader） |
