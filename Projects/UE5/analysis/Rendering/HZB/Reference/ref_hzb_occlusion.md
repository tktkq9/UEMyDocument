# ref: FViewOcclusionQueries / FOcclusionQueryVS / AllocateOcclusionTests

- 対象ファイル: `SceneOcclusion.h/.cpp`
- 概要: [[21_hzb_overview]]

---

## FViewOcclusionQueries（SceneOcclusion.h:22）

```cpp
struct FViewOcclusionQueries
{
    using FProjectedShadowArray = TArray<FProjectedShadowInfo const*, SceneRenderingAllocator>;
    using FPlanarReflectionArray = TArray<FPlanarReflectionSceneProxy const*, SceneRenderingAllocator>;
    using FRenderQueryArray = TArray<FRHIRenderQuery*, SceneRenderingAllocator>;

    // ─── クエリ対象の情報 ────────────────────────────────────────
    FProjectedShadowArray LocalLightQueryInfos;  // Local Light シャドウ
    FProjectedShadowArray CSMQueryInfos;         // CSM カスケード
    FProjectedShadowArray ShadowQueryInfos;      // 全シャドウ種別
    FPlanarReflectionArray ReflectionQueryInfos; // 平面反射

    // ─── RHI クエリオブジェクト ──────────────────────────────────
    FRenderQueryArray LocalLightQueries;
    FRenderQueryArray CSMQueries;
    FRenderQueryArray ShadowQueries;
    FRenderQueryArray ReflectionQueries;

    bool bFlushQueries = true; // true: RenderOcclusion でフラッシュ
};

// ビューごとのコンテナ
using FViewOcclusionQueriesPerView = TArray<
    FViewOcclusionQueries,
    TInlineAllocator<1, SceneRenderingAllocator>>;
```

---

## FOcclusionQueryVS（SceneOcclusion.h:47）

```cpp
class FOcclusionQueryVS : public FGlobalShader
{
    DECLARE_SHADER_TYPE(FOcclusionQueryVS, Global);

    // ─── シェーダーパラメータ ────────────────────────────────────
    // StencilingGeometryPosAndScale: バウンディング球の位置とスケール
    // FVector4f StencilingSpherePosAndScale[2]（ステレオレンダリング対応で 2 要素）

    void SetParameters(
        FRHIBatchedShaderParameters& BatchedParameters,
        const FViewInfo& View,
        const FSphere& BoundingSphere);

    // ─── 処理 ─────────────────────────────────────────────────
    // BoundingSphere から GStencilSphereVertexBuffer を変換
    // → 球の頂点を ClipSpace に投影して DrawCall
    // → PS なし（深度テストのみでサンプルカウント）
};
```

---

## AllocateOcclusionTests()（SceneOcclusion.h:42）

```cpp
void AllocateOcclusionTests(
    const FScene* Scene,
    FViewOcclusionQueriesPerView& QueriesPerView,
    TArrayView<class FViewInfo> Views,
    TArrayView<const class FVisibleLightInfo> VisibleLightInfos);

// InitViews() から呼ばれる
// → for each VisibleLight:
//     Local Light のシャドウに FRHIRenderQuery を確保して追加
//     CSM カスケードに FRHIRenderQuery を確保
// → for each PlanarReflection:
//     ReflectionQueries に追加
```

---

## HZBOcclusion テスト（HZBOcclusion.usf）

```
// ピクセルシェーダーでの HZB テスト処理概要

for each PrimitiveQuery:
    AABB.Min, AABB.Max = ProjectedAABBBounds
    UV_Min = ProjectedScreenUV(AABB.Min)
    UV_Max = ProjectedScreenUV(AABB.Max)

    // テストサイズに応じた Mip 選択
    Size = max(UV_Max - UV_Min) × HZBViewSize
    Mip = ceil(log2(Size))

    // 4コーナーをサンプル（最も遠い深度＝最小値）
    D0 = HZBTexture.SampleLevel(UV_Min, Mip)
    D1 = HZBTexture.SampleLevel(UV_Max, Mip)
    D2 = HZBTexture.SampleLevel(float2(UV_Max.x, UV_Min.y), Mip)
    D3 = HZBTexture.SampleLevel(float2(UV_Min.x, UV_Max.y), Mip)
    MaxDepth = max(D0, D1, D2, D3)  // ブロック内の最も遠い深度

    // AABB の最近深度と比較（Reverse-Z）
    AABB_MinDepth = ProjectedZ(AABB.Min)  // AABB の最も近い深度 = 最大Z値
    if (MaxDepth < AABB_MinDepth):
        // HZBブロックより AABB が手前にある → 可視
        Occluded = false
    else:
        Occluded = true
```

---

## RenderOcclusion() と PrimitiveVisibilityMap の更新

```
RenderOcclusion(GraphBuilder, Views, HZBTextures)
  │
  ├─ [A] HZBOcclusion.usf で GPU テスト
  │   → プリミティブごとの可視フラグを SampleResultBuffer に書き込み
  │
  ├─ [B] Shadow / Reflection クエリの DrawCall
  │   → FOcclusionQueryVS + 深度テストのみ
  │   → BeginRenderQuery / EndRenderQuery で囲む
  │
  ├─ [C] FlushOcclusionQueries()
  │   bFlushQueries=true の場合:
  │     → FRHIRenderQuery::GetResult() で結果を読み取り
  │     → VisibleSampleCount > 0 → 可視
  │
  └─ [D] FPrimitiveVisibilityMap 更新
      → FView::PrimitiveVisibilityMap[PrimIndex] = bVisible
      → 翌フレームの InitViews で DrawCall をスキップするために使用
      （1 フレーム遅延あり）
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.HZBOcclusion` | 1 | HZB Occlusion Culling 有効 |
| `r.OcclusionQueryLocation` | 0 | 0=BasePass 前, 1=後 |
| `r.AllowOcclusionQueries` | 1 | 全オクルージョンクエリ有効 |

---

## 関連リファレンス

- [[ref_hzb_resources]] — `FHZBParameters` / テクスチャ仕様
- [[b_hzb_occlusion]] — HZB Occlusion フロー詳細
