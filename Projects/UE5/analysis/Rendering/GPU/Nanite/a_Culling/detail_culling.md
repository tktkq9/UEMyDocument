# Nanite Culling シェーダー詳細

- グループ: a - Culling
- GPU 概要: [[01_nanite_gpu_overview]]
- CPU 詳細: [[a_nanite_cull_raster]]
- リファレンス: [[ref_culling]]

---

## 概要

Nanite のカリングパイプライン。GPU Driven で **プリミティブ → インスタンス → クラスター** の3段階にわたってカリングを実行し、  
最終的にラスタライズすべきクラスターリストを生成する。

**2パスオクルージョン構造：**
- **Pass 1（Main）**: 前フレームの HZB で保守的オクルージョンテスト → 大半のクラスターを確定
- **Pass 2（Post）**: Pass 1 で生成した新しい HZB で残りのオクルード候補を再テスト

---

## パス構成

```
[1] PrimitiveFilter（NanitePrimitiveFilter.usf）
     → HiddenPrimitivesList / ShowOnlyPrimitivesList でフィルタ
     → RWPrimitiveFilterBuffer（uint per primitive）

[2] Instance Hierarchy Culling（NaniteInstanceHierarchyCulling.usf）
     ├─ InstanceHierarchyCellChunkCull_CS  … セルチャンク単位の粗い分類
     └─ InstanceHierarchyChunkCull_CS      … インスタンスチャンク単位のカリング
     → インスタンス候補バッファ

[3] Cluster Culling Pass 1 — Main（NaniteClusterCulling.usf）
     └─ NodeAndClusterCull（CULLING_PASS=OCCLUSION_MAIN）
        ├─ NaniteHierarchyTraversal.ush#PersistentNodeAndClusterCull
        ├─ フラスタムカリング
        ├─ LOD 選択（画面上のクラスタ面積 → 誤差閾値）
        └─ HZB オクルージョン（前フレーム）
     → RWQueueState（可視クラスタ）/ RWOccludedInstances（再テスト候補）

[4] HZB Build（メインパイプライン → [7] 参照）

[5] Cluster Culling Pass 2 — Post（NaniteClusterCulling.usf）
     └─ NodeAndClusterCull（CULLING_PASS=OCCLUSION_POST）
        └─ 新しい HZB で再テスト → 可視なら Pass 2 ラスタライザへ
```

---

## PrimitiveFilter のコアロジック

```hlsl
[numthreads(64, 1, 1)]
void PrimitiveFilter(uint3 GroupId : SV_GroupID, uint GroupIndex : SV_GroupIndex)
{
    const uint PrimitiveId = GetUnWrappedDispatchThreadId(GroupId, GroupIndex, 64);

    // GPU Scene 上の IsHidden フラグを確認
    // HiddenPrimitivesList / ShowOnlyPrimitivesList を Binary Search で検索
    bool bHidden = IsNanitePrimitiveHidden(PrimitiveId, HiddenFilterFlags,
                       HiddenPrimitivesList, ShowOnlyPrimitivesList, ...);

    PrimitiveFilterBuffer[PrimitiveId] = bHidden ? 0u : 1u;
}
```

---

## Instance Hierarchy Culling のコアロジック

```hlsl
// セルチャンク単位（第1段階）
[numthreads(64, 1, 1)]
void InstanceHierarchyCellChunkCull_CS(uint3 GroupId, uint GroupThreadIndex)
{
    FCellChunkDraw CellDraw = ParentInfoBuffer[CellDrawIndex];
    // ItemChunksOffset + ChildIndex でチャンクを ProcessChunk() に渡す
    ProcessChunk(CellDraw.ItemChunksOffset + WorkGroupSetup.ChildIndex,
                 CellDraw.ViewGroupId, 0u);
}

// インスタンスチャンク単位（第2段階）
// Pass 2 時は InOccludedChunkDraws からオクルード候補を再テスト
[numthreads(64, 1, 1)]
void InstanceHierarchyChunkCull_CS(uint3 DispatchThreadID)
{
    // 各インスタンスの AABB を View 空間に変換 → フラスタムカリング
    // 可視インスタンス → VisibleInstances バッファに Append
}
```

---

## Cluster Culling（NodeAndClusterCull）のコアロジック

```hlsl
[numthreads(NANITE_PERSISTENT_CLUSTER_CULLING_GROUP_SIZE, 1, 1)]
void NodeAndClusterCull(uint GroupID, uint GroupIndex)
{
    // CULLING_TYPE に応じて3種の処理を切り替え
    // PERSISTENT_NODES_AND_CLUSTERS が主モード（1 Dispatch で全 BVH をカバー）
    PersistentNodeAndClusterCull<FNaniteTraversalClusterCullCallback>(
        GroupIndex, QueueStateIndex);
}
```

`FNaniteTraversalClusterCullCallback` が各ノード/クラスターで行う処理：
1. **フラスタムカリング**: AABB vs 6 Planes
2. **LOD 選択**: クラスタの投影面積 vs `LODError` 閾値 → 子ノードへ降りるか確定
3. **HZB オクルージョン**: クラスタ AABB を HZB でテスト（Pass1=前フレームHZB, Pass2=新HZB）
4. **可視クラスタの書き出し**: SW/HW フラグを付けて `VisibleClusters` バッファへ Append

---

## CULLING_PASS マクロの役割

| 値 | 定数名 | 動作 |
|---|--------|------|
| 0 | `CULLING_PASS_NO_OCCLUSION` | HZB なし（影パス等に使用） |
| 1 | `CULLING_PASS_OCCLUSION_MAIN` | 前フレーム HZB で Main オクルージョン |
| 2 | `CULLING_PASS_OCCLUSION_POST` | 新 HZB で Post（再テスト）|

---

## 入出力

### 入力

| リソース | 内容 |
|---------|------|
| `PrimitiveFilterBuffer` | PrimitiveFilter の結果 |
| `InstanceSceneData` | GPU Scene 上のインスタンス変換・バウンド |
| `HierarchyBuffer` | Nanite BVH ノードデータ |
| `ClusterPageData` | クラスターのローカル SDF・境界データ |
| `HZBTexture` | 前/新フレームの Hierarchical Z Buffer |
| `NaniteViews` | 各ビュー（メイン / CSM / ローカルライト）の変換行列 |

### 出力

| リソース | 内容 |
|---------|------|
| `PrimitiveFilterBuffer` | プリミティブ可視フラグ |
| `VisibleClustersSWHW` | SW / HW ラスタライザ向けの可視クラスターリスト |
| `OccludedInstances` | Pass 2 再テスト候補インスタンス |
| `QueueState` | 永続カリングキューの状態 |

---

## CPU 呼び出しの流れ

```
FNaniteCommandContext::DispatchSetupPipeline()     // NaniteCullRaster.cpp
  │
  ├─ AddPrimitiveFilterPass()
  │    └─ PrimitiveFilter CS ディスパッチ
  │
  ├─ Instance Hierarchy Culling ディスパッチ
  │    ├─ InstanceHierarchyCellChunkCull_CS
  │    └─ InstanceHierarchyChunkCull_CS
  │
  └─ IRenderer::DrawGeometry() → NodeAndClusterCull（Pass 1）
       │
       ├─ [HZB Build はここで実行]
       │
       └─ NodeAndClusterCull（Pass 2）
```
