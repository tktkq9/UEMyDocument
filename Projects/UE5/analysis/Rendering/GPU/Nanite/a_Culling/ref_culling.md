# REF: Nanite Culling シェーダー

- グループ: a - Culling
- 詳細: [[detail_culling]]
- CPU リファレンス: [[ref_nanite_cull_raster]]
- ソース: `Engine/Shaders/Private/Nanite/NanitePrimitiveFilter.usf`
          `Engine/Shaders/Private/Nanite/NaniteInstanceHierarchyCulling.usf`
          `Engine/Shaders/Private/Nanite/NaniteClusterCulling.usf`

---

## NanitePrimitiveFilter.usf

### PrimitiveFilter

```hlsl
[numthreads(64, 1, 1)]
void PrimitiveFilter(
    uint3 GroupId   : SV_GroupID,
    uint GroupIndex : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | HiddenPrimitivesList / ShowOnlyPrimitivesList を参照してプリミティブごとの可視フラグを設定 |
| **入力** | `NumPrimitives`, `HiddenFilterFlags`, `HiddenPrimitivesList`, `ShowOnlyPrimitivesList` |
| **出力** | `RWPrimitiveFilterBuffer`（uint per primitive: 0=hidden, 1=visible）|
| **CPU 関数** | `AddPrimitiveFilterPass()` |

---

## NaniteInstanceHierarchyCulling.usf

### InstanceHierarchyCellChunkCull_CS

```hlsl
[numthreads(64, 1, 1)]
void InstanceHierarchyCellChunkCull_CS(
    uint3 GroupId          : SV_GroupID,
    uint GroupThreadIndex  : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | インスタンス階層の第1レベル（Cell Chunk）を処理 — ワークグループローカルバランサで分散処理 |
| **入力** | `ParentInfoBuffer`（FCellChunkDraw）, `ViewGroupId` |
| **出力** | `AppendUncullableInstanceWork` への書き込み候補（後続 CS が読む）|
| **CPU 関数** | `DispatchSetupPipeline()` |

### InstanceHierarchyChunkCull_CS

```hlsl
[numthreads(64, 1, 1)]
void InstanceHierarchyChunkCull_CS(
    uint3 DispatchThreadID : SV_DispatchThreadID
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | インスタンスチャンク単位のフラスタムカリング（Pass1=全チャンク, Pass2=オクルード候補のみ）|
| **入力** | `InOccludedChunkDraws`（Pass2 時）, `InstanceSceneData` |
| **出力** | 可視インスタンスを `VisibleInstances` に Append |
| **CPU 関数** | `DispatchSetupPipeline()` |

### AppendUncullableInstanceWork

```hlsl
[numthreads(64, 1, 1)]
void AppendUncullableInstanceWork(uint3 DispatchThreadID : SV_DispatchThreadID)
```

| 目的 | カリング不可（常時可視）なインスタンスを可視リストに追加 |

### InstanceHierarchySanitizeInstanceArgsCS

```hlsl
[numthreads(1, 1, 1)]
void InstanceHierarchySanitizeInstanceArgsCS()
```

| 目的 | Indirect Dispatch 引数の安全チェック・上限クランプ |

---

## NaniteClusterCulling.usf

### NodeAndClusterCull

```hlsl
[numthreads(NANITE_PERSISTENT_CLUSTER_CULLING_GROUP_SIZE, 1, 1)]
void NodeAndClusterCull(
    uint GroupID    : SV_GroupID,
    uint GroupIndex : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Nanite BVH をトラバーサルしてフラスタム + LOD + HZB オクルージョンカリングを実行 |
| **パーミュテーション** | `CULLING_PASS`: 0=NoOcclusion, 1=Main, 2=Post; `CULLING_TYPE`: Nodes/Clusters/Persistent |
| **入力** | `HierarchyBuffer`, `ClusterPageData`, `HZBTexture`, `NaniteViews`, `QueueState` |
| **出力** | `RWVisibleClustersSWHW`（可視クラスター、SW/HW ビットフラグ付き）|
| **CPU 関数** | `IRenderer::DrawGeometry()` → `FInstanceHierarchyContext::RasterizeGeometry()` |

### CalculateSafeRasterizerArgs

```hlsl
[numthreads(1, 1, 1)]
void CalculateSafeRasterizerArgs()
```

| 目的 | SW/HW ラスタライザの Indirect Args がバッファ範囲外にならないよう上限チェック |
| 出力 | `OutSafeRasterizerArgsSWHW`（Indirect Dispatch/Draw 引数）|

### InitArgs

```hlsl
[numthreads(1, 1, 1)]
void InitArgs()
```

| 目的 | カリング開始前にキュー引数バッファを初期化 |

### InitClusterCullArgs

```hlsl
[numthreads(1, 1, 1)]
void InitClusterCullArgs()
```

| 目的 | クラスターカリング用 Indirect Dispatch 引数を設定 |

### InitNodeCullArgs

```hlsl
[numthreads(NANITE_MAX_CLUSTER_HIERARCHY_DEPTH + 1, 1, 1)]
void InitNodeCullArgs(uint GroupID : SV_GroupID, uint GroupIndex : SV_GroupIndex)
```

| 目的 | BVH ノードカリング用の深度別引数配列を初期化 |

### FeedbackStatus

```hlsl
[numthreads(1, 1, 1)]
void FeedbackStatus()
```

| 目的 | カリング統計（可視クラスタ数等）を CPU 読み取り可能バッファに書き出す（デバッグ用）|

---

## 共有ヘッダー

| ファイル | 役割 |
|---------|------|
| `NaniteCulling.ush` | `CULLING_PASS_*` / `CULLING_TYPE_*` 定数、`FNaniteTraversalClusterCullCallback` 定義 |
| `NaniteHierarchyTraversal.ush` | `PersistentNodeAndClusterCull()` 実装（BVH ループ本体）|
| `NaniteHierarchyTraversalCommon.ush` | AABB テスト / LOD 誤差計算等の共通ユーティリティ |
| `NaniteHZBCull.ush` | `IsOccluded()` — HZB に対する AABB テスト |
| `NaniteCullingCommon.ush` | フラスタム平面テスト、クラスター境界デコード |

---

> [!note]- 2パスオクルージョンの仕組みと恩恵
> Pass 1 では「前フレームの HZB」を使って保守的に多くのクラスターを棄却する。  
> ただし前フレーム HZB には現フレームの新しいオブジェクトが写っておらず、誤って棄却される可能性があるため、  
> Pass 2 では Pass 1 で書き込んだ VisBuffer を元に HZB を再構築し、オクルードされた候補を再テストすることで **False Negative を排除**する。
> この方式は Temporal な依存を持ちながらも1フレーム内で完結するため、Ghost アーティファクトが発生しない。

> [!note]- CULLING_TYPE の使い分け
> `CULLING_TYPE_PERSISTENT_NODES_AND_CLUSTERS` が主要モード（1 Dispatch でノードとクラスターを同時処理する Persistent Thread）。  
> `CULLING_TYPE_NODES` / `CULLING_TYPE_CLUSTERS` は特定の GPU や影パス等で Wave Occupancy を最大化するために使用する。  
> シェーダーは `CULLING_TYPE` プリプロセッサ分岐でコンパイル時に選択される。

> [!note]- AppendUncullableInstanceWork の役割
> 一部のインスタンス（例: カスタムカリングフラグ付き、またはビュー外だが Shadow Cast が必要なもの）は  
> フラスタムカリングをスキップして常に可視リストに追加される。  
> これにより VSM（Virtual Shadow Map）等の副次ビューに必要なインスタンスを取りこぼさない設計になっている。
