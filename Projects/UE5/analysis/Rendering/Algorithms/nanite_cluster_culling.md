---
name: Nanite Hierarchical Cluster Culling
description: Nanite の DAG 階層を用いた cluster カリング（永続スレッド + persistent cluster culling）の理論と UE 実装
type: project
---

# Nanite Hierarchical Cluster Culling（BVH 永続スレッドカリング）

- 上位: [[_algorithm_index]]
- 関連: [[nanite_visibility_buffer]] / [[nanite_sw_raster]] / [[nanite_lod]]
- 採用システム: Nanite の全ラスタパス（Primary / Shadow / Lumen Card Capture / VSM）
- 出典 ID: **S20**（[[_source_index]]）— §4 Cluster Hierarchy Traversal / §5 GPU-Driven Culling
- 影響元: Karis 2018 "Nanite Vision" 公式講演, Persistent Threads (Aila & Laine 2009)

---

## 1. 何のためのアルゴリズムか

Nanite は **数億ポリゴンのシーン** でも 1 億クラスタの中から **画面に映る数万クラスタ** だけを GPU で抽出する必要がある。CPU カリングは:

- メッシュ単位で精度が粗い（Frustum / Occlusion はインスタンス AABB のみ）
- インスタンス数が増えると CPU 律速

### 素朴な手法の問題

- **全クラスタ平面カリング compute**: シーン規模に比例するスレッド数 → スケールしない
- **ツリー再帰**: GPU 上での再帰は SIMT で発散しやすい
- **CPU BVH トラバーサル**: コア数で律速

### Nanite の貢献

- 各メッシュをオフラインで **DAG（Directed Acyclic Graph）** に組む（クラスタ → 親クラスタへの統合 LOD）
- ランタイムは **永続スレッド (Persistent Threads)** で BVH をトラバース
- **MainPass（前フレーム HZB で early reject）→ PostPass（今フレーム HZB で再評価）** の 2 パス occlusion culling
- Frustum / HZB / 画面サイズ（LOD）で再帰的に枝刈り

---

## 2. 理論

### 2.1 DAG クラスタ階層

- **Cluster**: 約 128 三角形のグループ、AABB / 法線錐 / LOD エラーを持つ
- **Cluster Group**: 隣接する 8〜32 クラスタを束ねた中間ノード
- **Hierarchy Node**: BVH の上位ノード、子は 8〜16 個（`NANITE_MAX_BVH_NODES_PER_GROUP`）
- DAG なので **複数 LOD で同じ三角形を共有可能**（連続 LOD の保証）

### 2.2 永続スレッドトラバーサル

通常の compute は「N スレッドを起動して 1 つずつ処理」だが、Nanite は:

```
- N スレッドを永続起動（GPU の SM 占有率に合わせた数）
- 各スレッドは Work Queue から候補ノードを pop
- ノードを評価 → 子をキューに push → ループ
- キューが空になれば終了
```

これにより BVH の深さや分岐が動的でも GPU が遊ばない。`CULLING_TYPE == NANITE_CULLING_TYPE_PERSISTENT_NODES_AND_CLUSTERS`（`NaniteClusterCulling.usf:73`）で永続モードに切替。

### 2.3 2 パス Occlusion Culling

Nanite は**前フレームの depth から構築した HZB**で early reject する設計（HZB はカメラ移動で誤判定する）:

| パス | 役割 | 使う HZB |
|------|------|---------|
| **MainPass** | 前フレーム可視と判定されたクラスタをラスタ | 前フレーム HZB |
| **PostPass** | MainPass で occluded 判定だったクラスタを **MainPass 結果で構築した今フレーム HZB** で再評価 | 今フレーム HZB |

`CULLING_PASS_NO_OCCLUSION` / `CULLING_PASS_OCCLUSION_MAIN` / `CULLING_PASS_OCCLUSION_POST` / `CULLING_PASS_EXPLICIT_LIST` の 4 種類が `NaniteCullRaster.cpp:37-40` に定義。

### 2.4 カリング判定

ノード / クラスタごとに以下を順に判定:

1. **Frustum Cull**: AABB がビューフラスタム外なら破棄
2. **Backface Cull (法線錐)**: クラスタの法線錐がカメラから完全裏向きなら破棄
3. **HZB Occlusion**: AABB を画面投影し、HZB の最大深度より深いなら破棄
4. **LOD Test**: §[[nanite_lod]] 参照（screen-space error）

DAG ノードは「自分の LOD で十分」なら子ノードに降りずクラスタを emit。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `NaniteCullRaster.cpp` | Cull/Raster オーケストレーション、CVar、PSO |
| `NaniteClusterCulling.usf` | クラスタ単位のカリング compute |
| `NaniteInstanceCulling.usf` | インスタンス単位の事前カリング |
| `NaniteInstanceHierarchyCulling.usf` | インスタンス階層カリング |
| `NaniteHierarchyTraversal.ush` | BVH 永続スレッドトラバーサル本体 |
| `NaniteHZBCull.ush` | HZB 占有テスト |
| `NaniteCullingCommon.ush` | カリング共通ユーティリティ |

### 3.2 候補ノード構造（`NaniteCullRaster.cpp:42-44`）

```cpp
static_assert(1 + NANITE_NUM_CULLING_FLAG_BITS + NANITE_MAX_INSTANCES_BITS <= 32, "FCandidateNode.x fields don't fit in 32bits");
static_assert(1 + NANITE_MAX_NODES_PER_PRIMITIVE_BITS + NANITE_MAX_VIEWS_PER_CULL_RASTERIZE_PASS_BITS <= 32, "FCandidateNode.y fields don't fit in 32bits");
static_assert(1 + NANITE_MAX_BVH_NODES_PER_GROUP <= 32, "FCandidateNode.z fields don't fit in 32bits");
```

候補ノードは 96-bit にビットパックされ、Work Queue に詰まれる。

### 3.3 永続スレッドカリング shader（`NaniteClusterCulling.usf:73-92`）

```hlsl
#if CULLING_TYPE == NANITE_CULLING_TYPE_PERSISTENT_NODES_AND_CLUSTERS
RWCoherentByteAddressBuffer  CandidateNodes;
RWCoherentByteAddressBuffer  CandidateClusters;
RWCoherentByteAddressBuffer  ClusterBatches;
RWCoherentByteAddressBuffer  InOutAssemblyTransforms;
#else
RWByteAddressBuffer          CandidateNodes;
RWByteAddressBuffer          CandidateClusters;
RWByteAddressBuffer          InOutAssemblyTransforms;
#endif

RWStructuredBuffer<FStreamingRequest> OutStreamingRequests;
RWByteAddressBuffer                   OutVisibleClustersSWHW;
RWBuffer<uint>                        VisibleClustersArgsSWHW;
```

`Coherent` バッファは GPU 内で全スレッドから可視性が保証され、Work Queue 共有に必要。

### 3.4 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.Nanite.AsyncRasterization` | 1 | カリング/SW ラスタを async compute で並列実行 |
| `r.Nanite.ComputeRasterization` | 1 | SW ラスタ（compute）有効化 |
| `r.Nanite.ProgrammableRaster` | 1 | プログラマブルラスタ（マテリアルラスタ）有効化 |
| `r.Nanite.MeshShaderRasterization` | 1 | HW ラスタで Mesh Shader を使用 |
| `r.Nanite.PrimShaderRasterization` | 1 | HW ラスタで Prim Shader を使用 |
| `r.Nanite.FilterPrimitives` | 1 | ビュー単位のプリミティブフィルタ |
| `r.Nanite.MaxPixelsPerEdge` | 1.0 | LOD 判定の三角形辺長予算 → [[nanite_lod]] |
| `r.Nanite.MinPixelsPerEdgeHW` | 32.0 | この辺長を超えたら HW ラスタへ送る |
| `r.Nanite.VSMInvalidateOnLODDelta` | 0 | ストリーミング待ちクラスタが VSM 無効化を起こす（実験的） |

### 3.5 ストリーミング要求

カリング中に「未ロードクラスタ」が必要な領域に来たら `OutStreamingRequests` に push（`NaniteClusterCulling.usf:89`）。CPU 側 `NaniteStreamingManager` が次フレームで該当クラスタをロード。

---

## 4. 近似・省略の差分

| 項目 | 理想 | UE 実装 | 影響 |
|------|------|--------|------|
| Occlusion 精度 | 完全 raycast | HZB（最大深度ピラミッド） | 細い前景で false-occluded（PostPass で救済） |
| HZB 鮮度 | 同フレーム | 前フレーム HZB → MainPass | カメラ急回転で 1 フレーム遅延 |
| BVH 並列度 | 完全 | 永続スレッド + Coherent バッファ | 同期コスト、Wave サイズ依存 |
| LOD 連続性 | 完全 | DAG（Cluster Group 単位の cut）| Group 境界で微小 popping 可 |
| Streaming 遅延 | 即時 | 1〜数フレーム遅延 | 高速移動でテンポラリ欠損（imposter で代替） |

---

## 5. パラメータと CVar

§3.4 にまとめ済み。

---

## 6. 代替手法との比較

| 手法 | 階層 | 精度 | GPU 効率 | UE 採用 |
|------|------|------|---------|--------|
| CPU Frustum Culling | インスタンス AABB | 粗 | CPU 律速 | UE Mesh Pass（標準） |
| GPU Instance Culling | インスタンス AABB + HZB | 中 | 良 | GPU Scene |
| **Nanite DAG + 永続スレッド** | **クラスタ DAG + 2 パス HZB** | **三角形級** | **億ポリゴンスケール** | **Nanite 専用** |
| Mesh Shader 単独 | メッシュレット | 三角形級 | プラットフォーム依存 | Nanite が併用（HW path） |
| Software Rasterizer GPU Pro 系 | クラスタ + occlusion | 中 | 中 | Nanite が拡張 |

### Mesh Shader vs Nanite

- Mesh Shader: GPU API レベルの仕組み、メッシュレット単位で頂点/プリミティブ生成
- Nanite: DAG + 永続スレッド + SW ラスタを組み合わせた**フレームワーク**。HW path として Mesh Shader/Prim Shader を使用するが、コア戦略は SW ラスタ + Cluster DAG

---

## 7. 参考資料

- [S20] Karis / Stenson / Sherlock 2021 SIGGRAPH §4-§5
- Aila & Laine 2009 "Understanding the Efficiency of Ray Traversal on GPUs" — Persistent Threads 原典
- 関連: [[nanite_visibility_buffer]] / [[nanite_sw_raster]] / [[nanite_lod]]

---

## 8. 相談用フック

- **理解度チェック**:
  - DAG が Tree でなく Graph な理由 → 連続 LOD で三角形共有のため
  - 2 パス HZB の必要性 → 前フレーム HZB の誤 occlusion 救済
  - 永続スレッドの利点 → BVH 深さ可変でも SM 遊ばせない
- **コード深掘り候補**:
  - `NaniteHierarchyTraversal.ush` の Work Queue pop/push 実装
  - `NaniteHZBCull.ush` の AABB→HZB tile 投影
  - `NaniteCullingCommon.ush` の Frustum/Cone カリング数式
- **未読箇所**:
  - S20 §5.3 GPU-Driven Streaming 詳細
  - `NaniteInstanceHierarchyCulling.usf` のシーンレベル階層
  - `NaniteFeedback.cpp`（フィードバック生成）
- **次の派生**:
  - Visibility Buffer 出力 → [[nanite_visibility_buffer]]
  - VSM への Nanite ラスタ → [[vsm_overview]]（未着手）
  - GPU Streaming → [[../Geometry/...]] 系（未着手）
