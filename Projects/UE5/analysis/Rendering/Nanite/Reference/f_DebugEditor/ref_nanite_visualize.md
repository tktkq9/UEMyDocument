# リファレンス：NaniteVisualize.h / NaniteVisualize.cpp

- グループ: f - Debug & Editor
- 上位: [[f_nanite_debug_editor]]
- 関連: [[ref_nanite_editor]] | [[ref_nanite_feedback]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteVisualize.h/.cpp`

---

## 概要

`r.Nanite.Visualize` CVar によって各種デバッグビューを提供するモジュール。  
クラスターID・マテリアルID・三角形密度・シェーダー複雑度等を色分け表示する。  
マウスカーソル下のクラスターをピッキングして詳細情報を表示する機能も持つ。

---

## 主要型・関数

```cpp
namespace Nanite
{

// デバッグビューモード（r.Nanite.Visualize の値に対応）
enum class EDebugViewMode : uint8
{
    None = 0,
    Wireframe,          // ワイヤーフレーム
    ClusterID,          // クラスター ID（色分け）
    GroupID,            // グループ ID
    PageID,             // ページ ID
    MaterialID,         // マテリアル ID
    ShadingBin,         // シェーディングビン番号
    RasterBin,          // ラスタライズビン番号
    TriangleCount,      // 三角形密度ヒートマップ
    LevelInstance,      // レベルインスタンス境界
    ShaderComplexity,   // シェーダー複雑度
    NaniteOverdraw,     // オーバードロー
    HierarchyOffset,    // BVH 階層オフセット
    // ...
};

// ビジュアライゼーションパスを RDG に追加
void AddVisualizationPasses(
    FRDGBuilder& GraphBuilder,
    const FScene& Scene,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    const FRasterResults& RasterResults,
    FRDGTextureRef OutputColor);

// ピッキング結果の表示（マウスカーソル位置のクラスター情報）
struct FNanitePickingFeedback
{
    uint32 ClusterIndex;
    uint32 TriangleIndex;
    uint32 MaterialIndex;
    FVector WorldPosition;
};

void DisplayPicking(
    FRDGBuilder& GraphBuilder,
    const FNanitePickingFeedback& PickingFeedback,
    FRDGTextureRef OutputColor);

// デバッグビューモードレンダリング
void RenderDebugViewMode(
    FRDGBuilder& GraphBuilder,
    EDebugViewMode DebugViewMode,
    const FViewInfo& View,
    const FRasterResults& RasterResults,
    FRDGTextureRef OutputColor);

} // namespace Nanite
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Nanite.Visualize` | 0 | デバッグビューモード番号（0=無効） |
| `r.Nanite.Picking` | 0 | ピッキング表示有効 |
