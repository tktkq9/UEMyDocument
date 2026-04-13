# D: 可視性カリング — BeginInitViews / HZB / ComputeRelevance

- 対象: `SceneVisibility.cpp`, `SceneRendering.h`
- 上位: [[23_scene_renderer_overview]]
- Reference: [[ref_view_info]] | [[ref_scene_renderer]]

---

## 概要

UE5 のレンダラーは毎フレーム「どのプリミティブをどのパスで描画するか」を  
以下の多段カリングで決定する。この処理は `BeginInitViews()` 〜 `EndInitViews()` の間に行われる。

```
1. 視錐台カリング（FrustumCull）
   └─ AABB vs 視錐台 6 平面で CPU 側判定

2. オクルージョンカリング（HZB Occlusion）
   └─ GPU の HZB（Hierarchical Z-Buffer）で前フレームの深度と比較

3. PrecomputedVisibility
   └─ 事前計算された静的可視性データ（BSP セル単位）

4. ComputeRelevance
   └─ 各プリミティブが「どのパス（GBuffer/Shadow/Translucent等）に参加するか」を確定
      → FPrimitiveViewRelevance を FViewInfo::PrimitiveViewRelevanceMap に格納

5. GatherDynamicMeshElements
   └─ ダイナミックプリミティブの FMeshBatch を FViewInfo::DynamicMeshElements に追加
```

---

## カリング段階の詳細

### 1. 視錐台カリング（FrustumCull）

```cpp
// SceneVisibility.cpp
void FSceneRenderer::FrustumCull(
    const FScene* Scene,
    FViewInfo& View)
{
    // FScene::PrimitiveBounds の球バウンドで視錐台テスト
    // PrimitiveVisibilityMap のビットを 1 に設定
    for (int32 Index : PotentiallyVisible)
    {
        if (View.ViewFrustum.IntersectBox(Bounds.BoxSphereBounds))
            View.PrimitiveVisibilityMap.AccessCorrespondingBit(Index) = true;
    }
}
```

### 2. HZB オクルージョン（前フレームの深度を利用）

```cpp
// 前フレームの SceneDepth から HZB を生成
// 現フレームのプリミティブの AABB を HZB に投影し、遮蔽判定
FHZBOcclusionTester OcclusionTester;
OcclusionTester.Submit(View, FrustumCulledPrimitives);
OcclusionTester.Retrieve(View);  // GPU 結果を CPU に回収（1 フレーム遅れ）
```

- 結果は 1 フレーム遅延（今フレームの判定は前フレームの HZB を使用）
- `r.HZBOcclusion = 0` で無効化可能

### 3. ComputeRelevance — パス参加フラグの確定

```cpp
// FPrimitiveSceneProxy::GetViewRelevance(View) を呼ぶ
// 返り値の FPrimitiveViewRelevance を PrimitiveViewRelevanceMap に格納
struct FPrimitiveViewRelevance
{
    uint8 bDrawRelevance      : 1;  // 描画対象か
    uint8 bStaticRelevance    : 1;  // SceneStaticMesh パス
    uint8 bDynamicRelevance   : 1;  // DynamicMesh パス
    uint8 bOpaque             : 1;  // 不透明パス
    uint8 bTranslucentSurface : 1;  // 半透明パス
    uint8 bShadowRelevance    : 1;  // シャドウパス
    uint8 bVelocityRelevance  : 1;  // ベロシティパス
};
```

---

## HZB（Hierarchical Z-Buffer）の生成

```
RenderPrepassAndVelocity()    DeferredShadingRenderer.cpp:2594
  └─ RenderPrePass()          :2384 — シーン深度を SceneTextures.Depth に書き込む
  
RenderOcclusion()
  └─ RenderHzb()              DeferredShadingRenderer.cpp:432 シグネチャ
      └─ BuildHZBFurthest()   — SceneDepth から Mip チェーンを生成
         BuildHZBClosest()    — 最近深度 HZB（オブジェクトごとの前面深度）
```

- HZB は前フレームの深度から生成されるため常に 1 フレーム遅延
- `FViewInfo::HZBMipmap` に保持され `FSceneViewState` にキャッシュされる

---

## GatherDynamicMeshElements

```cpp
// ComputeRelevance で bDynamicRelevance が true のプリミティブに対して
FPrimitiveSceneProxy::GetDynamicMeshElements(
    TArray<FViewInfo>& Views,
    TArray<FMeshBatchAndRelevance, SceneRenderingAllocator>& OutDynamicMeshElements);
// → FViewInfo::DynamicMeshElements に FMeshBatch を追加
```

`FMeshBatch` は頂点バッファ・インデックスバッファへのポインタと  
マテリアルへの参照を持つ 1 ドローコールの単位。

---

## コード実行フロー

### エントリポイント

```
FDeferredShadingSceneRenderer::BeginInitViews()        SceneVisibility.cpp
  │
  ├─ FrustumCull(Scene, View)
  │   └─ FScene::PrimitiveOctree を使って視錐台内のプリミティブを収集
  │      → PrimitiveVisibilityMap にビット設定
  │
  ├─ ComputeAndMarkRelevanceForViewParallel()            （並列タスク）
  │   └─ 各プリミティブの GetViewRelevance() を呼び出し
  │      → PrimitiveViewRelevanceMap を更新
  │
  ├─ BeginOcclusionTests()
  │   └─ HZBOcclusionTester.Submit() — GPU オクルージョンクエリ投入
  │
  └─ GatherDynamicMeshElements()
      └─ DynamicRelevance のプリミティブから FMeshBatch を収集
         → DynamicMeshElements に追加

FDeferredShadingSceneRenderer::EndInitViews()           SceneVisibility.cpp
  │
  ├─ HZBOcclusionTester.Retrieve()  — GPU 結果を回収（1フレーム遅れ）
  ├─ FinishInitDynamicShadows()     — シャドウ対象の確定
  └─ CreateUniformBuffers()         — ビューのユニフォームバッファ生成
```

### フロー詳細

1. **並列カリング（ParallelFor）**
   ```cpp
   // SceneVisibility.cpp
   ParallelFor(Views.Num(), [&](int32 ViewIndex)
   {
       FrustumCull(*Scene, Views[ViewIndex]);
       ComputeAndMarkRelevanceForViewParallel(..., Views[ViewIndex]);
   });
   ```
   - `BeginInitViews()` はタスクグラフで並列実行される
   - `ERDGBuilderFlags::ParallelSetup` と連動

2. **InitViews の完了タイミング**
   ```
   BeginInitViews()  :2052  → タスク起動のみ（非ブロッキング）
   ...（他の非同期処理）
   EndInitViews()    :2316  → タスク完了を待機し DynamicMeshElements を確定
   ```
   - `BeginInitViews()` と `EndInitViews()` の間に GPU Scene アップロードや  
     Shadow 初期化など別タスクが走ることで並列化される

3. **HZB の生成タイミング**
   ```
   RenderPrePass()          深度バッファを埋める
     └─ RenderOcclusion()
         └─ RenderHzb()     SceneDepth から HZB Mip チェーンを生成
                            生成された HZB は FSceneViewState に保存
                            → 次フレームのオクルージョンカリングに使用
   ```

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|--------|------|
| `FDeferredShadingSceneRenderer::BeginInitViews()` | `SceneVisibility.cpp` | 可視性計算開始エントリ |
| `FDeferredShadingSceneRenderer::EndInitViews()` | `SceneVisibility.cpp` | 可視性計算完了・DrawList 確定 |
| `FSceneRenderer::FrustumCull()` | `SceneVisibility.cpp` | 視錐台カリング |
| `ComputeAndMarkRelevanceForViewParallel()` | `SceneVisibility.cpp` | 並列 Relevance 計算 |
| `FHZBOcclusionTester` | `HZBOcclusion.h` | HZB オクルージョンクエリ管理 |
| `FDeferredShadingSceneRenderer::RenderHzb()` | `DeferredShadingRenderer.cpp` | HZB Mip チェーン生成 |
| `FPrimitiveViewRelevance` | `PrimitiveViewRelevance.h` | [[ref_view_info]] パス参加フラグ |
| `FViewInfo::DynamicMeshElements` | `SceneRendering.h` | [[ref_view_info]] 動的 DrawList |
