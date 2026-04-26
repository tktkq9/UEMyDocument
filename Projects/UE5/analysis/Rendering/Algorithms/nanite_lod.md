---
name: Nanite LOD Selection (Screen-Space Error)
description: DAG クラスタを screen-space error で連続 LOD 選択するアルゴリズムと UE 実装
type: project
---

# Nanite LOD 選択（Screen-Space Error 連続 LOD）

- 上位: [[_algorithm_index]]
- 関連: [[nanite_cluster_culling]] / [[nanite_visibility_buffer]] / [[nanite_sw_raster]]
- 採用システム: Nanite の全カリングパス（クラスタ階層トラバース時）
- 出典 ID: **S20**（[[_source_index]]）— §2 Cluster Hierarchy / §4 Cut Selection
- 影響元: Hoppe 1996 "Progressive Meshes", Cignoni et al. 2003 "BDAM Adaptive Mesh"

---

## 1. 何のためのアルゴリズムか

「視線距離・画面サイズに応じた適切なポリゴン密度」を**ビューごと、フレームごと**に選ぶ。Nanite は数億ポリゴンを扱うので:

- 静的 LOD（N 段階）では切替時に **popping**
- 距離単独ではビュー（FoV / 解像度 / 角度）変化に追従不能
- 画面サイズ < 1 ピクセルの三角形は完全な無駄

### 素朴な手法の問題

- **離散 N 段 LOD**: 切替で popping 顕著
- **CPU で screen-space error 計算**: メッシュ数で律速
- **Continuous LOD（PM）**: 連続だが GPU 直接トラバース不向き

### Nanite の貢献

- **DAG（Cluster Group 単位の Cut）** で連続 LOD を実現（Group 内は同 LOD で統一されるので連続性保証）
- ランタイムで **cluster 単位の screen-space error** を評価し、`MaxPixelsPerEdge` 予算と比較
- 親 Group の error が予算超 → 子 Group に降りる、未満 → そのクラスタを採用

---

## 2. 理論

### 2.1 Cluster Group / Cut 概念

DAG では:
- 親 Group (粗) ↔ 子 Group (細) が連続 LOD 階層を構成
- フレームごとに「親 / 子のどちらを採用するか」の **Cut（境界線）** を引く
- Cut は同じ Cluster Group 内で必ず一致 → 三角形の重複や穴が発生しない

### 2.2 Screen-Space Error 計算

各 Cluster / Cluster Group は**保守的な error 半径**（オフライン算出）を持つ:
- `LODError`: 子 LOD と親 LOD の最大ジオメトリ誤差（ワールド空間距離）
- `BoundingSphere`: クラスタ AABB の外接球

ランタイムで:

```
projectedError = LODError * (screenHeight / (2 * distance * tan(FoV/2)))
                = LODError * pixelsPerWorldUnit
```

`projectedError > MaxPixelsPerEdge` なら子 Group へ降りる。

`r.Nanite.MaxPixelsPerEdge = 1.0` がデフォルト → 「画面上で 1 ピクセル誤差以下になるまで細分化」。

### 2.3 LOD 評価の階層化

実装では BVH トラバース中に各ノードで:

1. ノード全体の error 球を画面投影
2. 投影サイズが `MaxPixelsPerEdge` 未満 → そのまま採用（imposter / 親クラスタ emit）
3. 超過 → 子ノード（より細分化）をキューに push
4. リーフ（クラスタ）まで到達 → そのクラスタを VisibleClusters に emit

`NaniteHierarchyTraversal.ush`（[[nanite_cluster_culling]] §3.1）で実装。

### 2.4 Imposter Path

非常に小さいインスタンス（< `r.Nanite.ImposterMaxPixels = 5` ピクセル）は最低 LOD ですら過剰 → **2D 板（imposter）** で代替:

```cpp
static TAutoConsoleVariable<int32> CVarNaniteImposterMaxPixels(
    TEXT("r.Nanite.ImposterMaxPixels"),
    5,
    TEXT("The maximum size of imposters measured in pixels."));
```

オフライン生成された billboard アトラスを参照。

### 2.5 Dynamic Resolution Scaling

ビュー予算超過時に `MaxPixelsPerEdge` を動的に増やす（粗くする）:
- `r.Nanite.PrimaryRaster.PixelsPerEdgeScaling`（Primary パス用）
- `r.Nanite.ShadowRaster.PixelsPerEdgeScaling`（Shadow パス用）

`NaniteCullRaster.cpp:341-352` 抜粋:
```cpp
// i.e. if r.Nanite.MaxPixelsPerEdge is 1.0 and r.Nanite.PrimaryRaster.PixelsPerEdgeScaling is 20%,
// when heavily over budget r.Nanite.MaxPixelsPerEdge will be scaled to 5.0
```

### 2.6 Tessellation との連動

`r.Nanite.DicingRate = 2.0` がテッセレーション後の micropolygon 目標。`InvDiceRate = MaxPixelsPerEdge / DicingRate`（`NaniteCullRaster.cpp:6560`）でディスプレースメント評価:

```cpp
UniformParameters->InvDiceRate = CVarNaniteMaxPixelsPerEdge.GetValueOnRenderThread()
                               / CVarNaniteDicingRate.GetValueOnRenderThread();
```

### 2.7 Displacement Fade

ディスプレースメント（高さマップ）は近距離だけ有効化、遠距離はフラットに戻す（コスト節約）:

```cpp
// NaniteCullRaster.cpp:3057-3071
static void CalcDisplacementFadeSizes(const FDisplacementFadeRange& Range,
                                       float& FadeSizeStart, float& FadeSizeStop)
{
    const float EdgesPerPixel = 1.0f / CVarNaniteMaxPixelsPerEdge.GetValueOnRenderThread();
    if (!Range.IsValid())
    {
        FadeSizeStart = FadeSizeStop = 0.0f;
    }
    else
    {
        FadeSizeStop  = EdgesPerPixel * FMath::Max(Range.EndSizePixels, UE_KINDA_SMALL_NUMBER);
        FadeSizeStart = EdgesPerPixel * FMath::Max(Range.StartSizePixels, Range.EndSizePixels + UE_KINDA_SMALL_NUMBER);
    }
}
```

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `NaniteCullRaster.cpp` | LOD CVar 定義、Imposter / Displacement fade |
| `NaniteCullingCommon.ush` | screen-space error 評価関数 |
| `NaniteHierarchyTraversal.ush` | DAG 走査と Cut 決定 |
| `NaniteHZBCull.ush` | HZB と LOD の組合わせ判定 |
| `NaniteResources.h`（Engine モジュール） | クラスタ LODError / BoundingSphere 永続化 |

### 3.2 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.Nanite.MaxPixelsPerEdge` | 1.0 | 三角形辺長予算（小 = 高品質） |
| `r.Nanite.MinPixelsPerEdgeHW` | 32.0 | SW/HW 切替閾値 → [[nanite_sw_raster]] |
| `r.Nanite.ImposterMaxPixels` | 5 | imposter 切替閾値 |
| `r.Nanite.DicingRate` | 2.0 | テッセレーション目標 micropolygon |
| `r.Nanite.MaxPatchesPerGroup` | 5 | テッセレーションパッチ/group |
| `r.Nanite.PrimaryRaster.PixelsPerEdgeScaling` | — | 動的解像度時の粗化下限 % |
| `r.Nanite.ShadowRaster.PixelsPerEdgeScaling` | — | シャドウパス用粗化下限 % |
| `r.Nanite.DepthBucketing` | 1 | 深度バケット振り分け |
| `r.Nanite.DepthBucketsMinZ` | 1000 | バケット最小 Z |
| `r.Nanite.DepthBucketsMaxZ` | 100000 | バケット最大 Z |
| `r.Nanite.VSMInvalidateOnLODDelta` | 0 | LOD 不一致で VSM 無効化（実験的） |

### 3.3 EdgesPerPixel 計算（`NaniteCullRaster.cpp:3060`）

```cpp
const float EdgesPerPixel = 1.0f / CVarNaniteMaxPixelsPerEdge.GetValueOnRenderThread();
```

LOD 判定や displacement fade で「1 ピクセルあたりの三角形辺数」として使用。

### 3.4 Streaming と LOD のずれ

ストリーミング遅延で「目標 LOD のクラスタが未ロード」だと粗いクラスタで代用 → VSM cache 不整合の可能性。`r.Nanite.VSMInvalidateOnLODDelta = 1` で目標 LOD 一致まで invalidation 強制（実験的）。

---

## 4. 近似・省略の差分

| 項目 | 理想 | UE 実装 | 影響 |
|------|------|--------|------|
| 連続性 | 完全連続 | Cluster Group 単位 Cut | Group 境界で微小 popping（典型 unnoticeable） |
| Error 評価 | 完全（ピクセル毎） | 保守球 + screen 投影 | 過大評価で過剰細分化（保守的） |
| 動的形状 | 都度再 DAG | 静的 DAG（オフライン） | スキニング/WPO は事前評価 LOD 流用 |
| Displacement | 全範囲 | Fade Range で遠距離オフ | 中距離での微細失われ |
| 動的解像度 | 完全追従 | `MaxPixelsPerEdge` スケール | 重い時は粗化のみ（細密化はなし） |

---

## 5. パラメータと CVar

§3.2 にまとめ済み。`r.Nanite.MaxPixelsPerEdge=2` で半分の精度（軽量化）、`0.5` で倍精度（重い）。

---

## 6. 代替手法との比較

| 手法 | 連続性 | GPU 評価 | UE 採用 |
|------|-------|---------|--------|
| 離散 LOD（UE 標準 SM） | 不連続（fade で対処） | CPU | 非 Nanite メッシュ |
| Progressive Mesh (Hoppe 1996) | 連続 | CPU/GPU 限定 | 古典 |
| Continuous LOD Quadtree（地形 BDAM） | 連続 | GPU 可 | Landscape 系 |
| **Nanite DAG + Group Cut** | **クラスタ Group 単位連続** | **GPU 完全** | **Nanite 標準** |
| Mesh Shader Meshlet LOD | 中 | GPU | DX12 Mesh Shader 系 |

### Hoppe PM vs Nanite DAG

- Hoppe PM: 1 つの線形 edge collapse 履歴 → 任意中間 LOD
- Nanite: 局所単位（Cluster Group）で並列 LOD 選択可能 → GPU 親和性高

---

## 7. 参考資料

- [S20] Karis / Stenson / Sherlock 2021 §2, §4
- Hoppe 1996 "Progressive Meshes"
- Cignoni et al. 2003 "BDAM — Batched Dynamic Adaptive Meshes for High Performance Terrain Visualization"
- 関連: [[nanite_cluster_culling]] / [[nanite_sw_raster]] / [[nanite_visibility_buffer]]

---

## 8. 相談用フック

- **理解度チェック**:
  - DAG が連続 LOD を保証する仕組み → Cluster Group 単位で Cut を引くため隣接クラスタ間に隙間/重複が出ない
  - `MaxPixelsPerEdge` の意味 → 1 三角形辺が画面で何ピクセル占めるかの目標
  - Imposter の役割 → 数ピクセル以下のインスタンスでクラスタトラバース不要化
- **コード深掘り候補**:
  - `NaniteCullingCommon.ush` の screen-space error 評価関数本体
  - `NaniteHierarchyTraversal.ush` の Cut 決定ロジック
  - `NaniteResources.h`（Engine モジュール）の `FCluster::LODError` 等
- **未読箇所**:
  - S20 §2.5 DAG ビルド時の error 半径計算
  - Spline Mesh / Skinned Nanite の動的 LOD 補正
  - Nanite Imposter 生成パス（オフライン）
- **次の派生**:
  - Tessellation との連動 → [[nanite_tessellation]]（未着手）
  - VSM 不整合と LOD → [[vsm_overview]]（未着手）
  - DAG オフラインビルド → [[../Geometry/...]] 系
