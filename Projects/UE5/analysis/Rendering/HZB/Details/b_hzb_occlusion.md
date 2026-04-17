# b: HZB Occlusion Culling

- 対象ファイル: `SceneOcclusion.h/.cpp`
- 概要: [[21_hzb_overview]]

---

## 概要

HZB を使った GPU Occlusion Query は**1フレーム遅延**でプリミティブの可視判定を行う。  
前フレームの HZB に各プリミティブの AABB を投影し、遮蔽されていれば次フレームで描画をスキップする。  
`FViewOcclusionQueries` がクエリを管理し、`RenderOcclusion()` で結果を Readback する。

---

## FViewOcclusionQueries（SceneOcclusion.h）

```cpp
struct FViewOcclusionQueries
{
    using FProjectedShadowArray = TArray<FProjectedShadowInfo const*, SceneRenderingAllocator>;
    using FPlanarReflectionArray = TArray<FPlanarReflectionSceneProxy const*, SceneRenderingAllocator>;
    using FRenderQueryArray = TArray<FRHIRenderQuery*, SceneRenderingAllocator>;

    // 各種クエリ対象の情報
    FProjectedShadowArray LocalLightQueryInfos;   // Local Light シャドウ
    FProjectedShadowArray CSMQueryInfos;          // CSM シャドウ
    FProjectedShadowArray ShadowQueryInfos;       // 全シャドウ
    FPlanarReflectionArray ReflectionQueryInfos;  // 平面反射

    // RHI クエリオブジェクト
    FRenderQueryArray LocalLightQueries;
    FRenderQueryArray CSMQueries;
    FRenderQueryArray ShadowQueries;
    FRenderQueryArray ReflectionQueries;

    bool bFlushQueries = true;  // 結果を即時 Flush するか
};

using FViewOcclusionQueriesPerView = TArray<FViewOcclusionQueries, TInlineAllocator<1, SceneRenderingAllocator>>;
```

---

## FOcclusionQueryVS（SceneOcclusion.h）

```cpp
class FOcclusionQueryVS : public FGlobalShader
{
    // 遮蔽判定用のボックスを描画する頂点シェーダー
    // StencilingGeometryPosAndScale: 投影球/ボックスのスケールと位置

    void SetParameters(
        FRHIBatchedShaderParameters& BatchedParameters,
        const FViewInfo& View,
        const FSphere& BoundingSphere)
    {
        // BoundingSphere から StencilingSpherePosAndScale を計算
        // → GStencilSphereVertexBuffer を変換して描画
    }
};
```

---

## AllocateOcclusionTests() → RenderOcclusion() フロー

```
InitViews():
  AllocateOcclusionTests(Scene, QueriesPerView, Views, VisibleLightInfos)
    │
    ├─ for each LocalLight in VisibleLightInfos:
    │   → FRHIRenderQuery を確保して LocalLightQueries に追加
    │
    ├─ for each CSM Cascade:
    │   → CSMQueries に追加
    │
    └─ for each PlanarReflection:
        → ReflectionQueries に追加

[前フレームの HZB を使って GPU Occlusion]
RenderOcclusion(GraphBuilder, Views, ...)
  │
  ├─ [A] HZBOcclusion.usf による AABB テスト
  │   → for each Primitive:
  │       プリミティブの AABB を FurthestHZB に投影
  │       AABB の全 8 頂点を ProjectedUV に変換
  │       → HZB.SampleLevel(ProjectedUV, MipLevel) × 4 隅でサンプル
  │       → MaxHZBDepth < AABB最近深度 → Occluded（遮蔽）
  │       → 結果を SampleResultBuffer に書き込み
  │
  ├─ [B] RHI Occlusion Query（Bounding Box Draw）
  │   → FOcclusionQueryVS + 深度テストのみの PS なし DrawCall
  │   → GPU が描画される（＝可視）ピクセル数をカウント
  │   → BeginRenderQuery / EndRenderQuery で囲む
  │
  ├─ [C] FlushOcclusionQueries()
  │   → GPU 結果を CPU に Readback
  │   → FRHIRenderQuery::GetResult() で可視ピクセル数取得
  │
  └─ [D] FPrimitiveVisibilityMap 更新
      Primitive.PrimitiveVisibilityId → bVisible フラグ
      → 翌フレームの InitViews() で DrawCall をスキップ
      （1 フレーム遅延 = 急に現れるオブジェクトが 1 フレーム欠ける可能性）
```

---

## 1フレーム遅延の仕組み

```
Frame N:
  → HZB を構築（SceneDepth から）
  → RenderOcclusion() でクエリ発行（前フレームの HZB を参照）
  → GPU コマンドは非同期実行

Frame N+1 の InitViews():
  → FlushOcclusionQueries() で前フレームのクエリ結果を取得
  → PrimitiveVisibilityMap を更新
  → 不可視プリミティブは DrawCall を発行しない

問題点:
  → カメラ急旋回時に可視オブジェクトが 1 フレーム消える（ポッピング）
  → r.HZBOcclusion=0 で無効化可能
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.HZBOcclusion` | 1 | HZB Occlusion Culling 有効 |
| `r.OcclusionQueryLocation` | 0 | 0=BasePass前, 1=BasePass後 |
| `r.MinScreenRadiusForDepthPrepass` | 0.03 | Depth Prepass の最小スクリーン半径 |

---

## 関連リファレンス

- [[ref_hzb_occlusion]] — `FViewOcclusionQueries` / `FOcclusionQueryVS` 詳細
- [[a_hzb_build]] — BuildHZB() ミップチェーン生成
