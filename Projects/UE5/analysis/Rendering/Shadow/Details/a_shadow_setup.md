# a: Shadow セットアップ（FProjectedShadowInfo 生成）

- 対象ファイル: `ShadowSetup.cpp` / `ShadowRendering.h`
- 概要: [[17_shadow_overview]]

---

## 概要

`InitViews()` フェーズで `ShadowSetup.cpp` がシャドウを種別ごとに分類し、  
`FProjectedShadowInfo` を生成する。この構造体が Shadow Depth 描画から  
Projection まで一貫して使い回される。

---

## Shadow 種別と生成関数

| 種別 | 生成関数 | 対象 |
|-----|---------|-----|
| Whole Scene / CSM | `CreateWholeSceneProjectedShadow()` | Directional Light のカスケード |
| Per-Object Dynamic | `CreatePerObjectProjectedShadow()` | 動的オブジェクトの個別シャドウ |
| Spot Light | `CreateSpotLightProjectedShadow()` | Spot Light のコーン錐台シャドウ |
| Point Light | `CreatePointLightProjectedShadow()` | Point Light のキューブ 6 面シャドウ |
| Cached Whole Scene | `CreateCachedWholeSceneProjectedShadow()` | 静的ジオメトリのキャッシュシャドウ |
| Translucency | `CreateTranslucentShadow()` | 半透明オブジェクトのシャドウ |

---

## Shadow アトラスパッキング

複数の `FProjectedShadowInfo` を1枚のアトラステクスチャにパッキングする。

```
SortShadowInfo()                                ShadowSetup.cpp
  → 解像度降順でソート（大きいシャドウを先に配置）

ShadowDepthAtlas のレイアウト:
  ┌──────────┬────────┐
  │ CSM[0]   │ CSM[1] │
  │ 1024x1024│512x512 │
  ├────────────────────┤
  │ Spot[0]  │ Spot[1]│
  │ 256x256  │256x256 │
  └──────────┴────────┘

FProjectedShadowInfo.X, Y → アトラス内オフセット
FProjectedShadowInfo.ResolutionX/Y → このシャドウのサイズ
FProjectedShadowInfo.BorderSize → PCF フィルタリング用のボーダー幅
```

---

## SetupMeshDrawCommandsForShadowDepth() フロー

```
InitViews()
  └─ SetupMeshDrawCommandsForShadowDepth()      ShadowSetup.cpp
      │
      ├─ for each FLightSceneInfo in VisibleLights:
      │
      │   ── Directional Light（CSM）──────────────────────
      │   CreateWholeSceneShadowCascades()
      │     for CascadeIndex in [0, MaxCascades):
      │       FProjectedShadowInfo* Shadow = new FProjectedShadowInfo()
      │       Shadow->SetupWholeSceneProjection(
      │           LightSceneInfo, View, CascadeSettings, ...)
      │       → ShadowBounds / TranslatedWorldToClip を計算
      │       → FVisibleLightInfo::AllProjectedShadows に追加
      │
      │   ── Spot / Point Light ─────────────────────────
      │   CreateSpotLightProjectedShadow() / CreatePointLightProjectedShadow()
      │     → FProjectedShadowInfo::SetupSpotProjection(...)
      │     → FProjectedShadowInfo::SetupCubemapProjection(...)（Point 6面）
      │
      │   ── Per-Object Dynamic Shadow ───────────────────
      │   for each 動的プリミティブ:
      │     CreatePerObjectProjectedShadow()
      │       → オブジェクトの AABB から ShadowBounds を計算
      │       → 必要な解像度を MaxScreenPercent から算出
      │
      └─ AttemptToPackShadowsInAtlas()
          → SortShadowInfo() でソート
          → アトラステクスチャに座標 (X, Y) を割り当て
          → FProjectedShadowInfo.RenderTargets を設定
```

---

## FProjectedShadowInfo の主要メンバ

```cpp
// ShadowRendering.h:278
class FProjectedShadowInfo : public FRefCountedObject
{
    FViewInfo* ShadowDepthView;       // Shadow Depth 描画用ビュー（ライスト視点）
    FShadowMapRenderTargets RenderTargets; // Shadow Depth Atlas テクスチャ
    EShadowDepthCacheMode CacheMode;  // キャッシュモード（Static/Dynamic）
    FMatrix TranslatedWorldToView;    // ライスト View 行列
    FMatrix44f TranslatedWorldToClipInnerMatrix; // ライスト ViewProj 行列
    float InvMaxSubjectDepth;         // 深度正規化係数
    float MaxSubjectZ;                // 遮蔽物の最大深度
    float MinSubjectZ;                // 遮蔽物の最小深度
    uint32 X, Y;                      // アトラス内位置
    uint32 ResolutionX, ResolutionY;  // Shadow Map サイズ
    uint32 BorderSize;                // PCF ボーダー幅
    FShadowCascadeSettings CascadeSettings; // CSM カスケード設定
    FSphere ShadowBounds;             // シャドウ影響球
};
```

---

## 関連リファレンス

- [[ref_projected_shadow_info]] — `FProjectedShadowInfo` 全メンバ詳細
- [[b_shadow_depth]] — Shadow Depth Map 描画
- [[c_shadow_projection]] — Shadow Projection（PCF/PCSS）
