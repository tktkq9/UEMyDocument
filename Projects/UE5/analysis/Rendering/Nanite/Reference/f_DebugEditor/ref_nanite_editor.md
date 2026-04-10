# リファレンス：NaniteEditor.h / NaniteEditor.cpp

- グループ: f - Debug & Editor
- 上位: [[f_nanite_debug_editor]]
- 関連: [[ref_nanite_visualize]] | [[ref_nanite_materials_scene_ext]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteEditor.h/.cpp`

---

## 概要

エディタでの Nanite プリミティブ操作を支援するモジュール。  
- ヒットプロキシ（クリック判定用テクスチャ）の生成
- 選択オーバーレイ（選択中プリミティブのハイライト）
- レベルインスタンス境界の可視化

ランタイムビルドには含まれず、`WITH_EDITOR` ガードで囲われる。

---

## 主要関数

```cpp
namespace Nanite
{

// ヒットプロキシ描画（クリック判定用テクスチャに各プリミティブの ID を書き込む）
void DrawHitProxies(
    FRDGBuilder& GraphBuilder,
    const FScene& Scene,
    const FViewInfo& View,
    FRasterResults& RasterResults,
    FRDGTextureRef HitProxyTexture,
    FRDGTextureRef HitProxyDepthTexture);

// 現在選択中のヒットプロキシ ID を SRV で返す（カリングシェーダーが参照）
FRHIShaderResourceView* GetEditorSelectedHitProxyIdsSRV(const FScene& Scene);

// エディタ選択ハイライト描画
void DrawEditorSelection(
    FRDGBuilder& GraphBuilder,
    const FScene& Scene,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    FRDGTextureRef SelectionOutlineTexture);

// レベルインスタンスの境界・ハイライト可視化
void DrawEditorVisualizeLevelInstance(
    FRDGBuilder& GraphBuilder,
    const FScene& Scene,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    FRDGTextureRef OutputTexture);

} // namespace Nanite
```

---

## GPUStat

| 統計名 | 説明 |
|--------|------|
| `NaniteEditor` | エディタ描画パス全体の GPU 時間 |
