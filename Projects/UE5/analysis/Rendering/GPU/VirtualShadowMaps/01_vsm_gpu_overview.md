# Virtual Shadow Map GPU 処理概要

- グループ: VSM GPU
- CPU 概要: [[04_vsm_overview]]
- CPU 詳細: [[a_vsm_array]] | [[b_vsm_cache_manager]] | [[c_vsm_clipmap]] | [[d_vsm_projection]]

---

## VSM GPU パイプライン実行順

Virtual Shadow Map は「仮想ページテーブル＋物理ページアトラス」を用いて、  
必要なシャドウページのみを差分更新する Shadow Mapping システム。

```
[フレーム前処理]
[1]  Cache GPU Invalidation
     ├─ VSMUpdateViewInstanceStateCS … インスタンスの動的/静的状態を更新
     ├─ ProcessInvalidationQueueGPUCS … 変化したインスタンスの VSM ページを無効化
     └─ InvalidateInstancePagesLoadBalancerCS … ページ無効化を負荷分散実行

[ページ要求フェーズ]
[2]  Page Marking
     ├─ PruneLightGridCS … ライトグリッドから VSM を持つライトを抽出
     ├─ GeneratePageFlagsFromPixels … GBuffer / Hair / 水面ピクセルからページ要求フラグを生成
     ├─ GeneratePageFlagsFromFroxelsCS … Froxel（スポット/ポイントライト半球）からページ要求
     └─ MarkCoarsePages … ディレクショナルライト粗解像度ページのフラグ立て

[ページ割り当てフェーズ]
[3]  Page Management
     ├─ GenerateHierarchicalPageFlags … ページフラグを MIP 階層に伝播（親ページもマーク）
     └─ PropagateMappedMips … 割り当て済みページを上位 MIP に伝播

[4]  Physical Page Management
     ├─ InitPageRectBounds … ページ矩形バウンドを初期化
     ├─ UpdatePhysicalPageAddresses … 仮想ページ → 物理ページアドレスのテーブルを更新
     ├─ UpdatePhysicalPages … 物理ページの割り当て／解放を処理
     ├─ ClearPageTableCS … ページテーブルのクリア
     ├─ AllocateNewPageMappingsCS … 新規ページのマッピングを確保
     ├─ PackAvailablePages … 空き物理ページリストのパック
     └─ AppendPhysicalPageLists … 物理ページリストに追記

[シャドウ描画フェーズ]
[5]  Compact Views
     └─ CompactViewsVSM_CS … 描画が必要な VSM ビューをコンパクト化

[6]  Build Per-Page Draw Commands
     ├─ CullPerPageDrawCommandsCs … インスタンスをページ単位でカリング・Draw Command 生成
     ├─ AllocateCommandInstanceOutputSpaceCs … Draw Command のインスタンス出力バッファ確保
     ├─ OutputCommandInstanceListsCs … インスタンスリストをソート済み出力バッファに書き出し
     └─ ComputeExplicitChunkDrawsViewMask … Explicit チャンク Draw の ビューマスク計算

[7]  Shadow Depth Rendering（CPU Driven）
     ├─ Nanite path: Nanite::IRenderer（NaniteClusterCulling + NaniteRasterizer）
     └─ 非 Nanite: Shadow Depth Pass（ShadowDepth.usf）— [6] の Draw Commands を使用

[シャドウ投影フェーズ]
[8]  Projection（SMRT サンプリング）
     └─ VirtualShadowMapProjection CS … SceneDepth から VSM をサンプリング → ShadowMask 生成

[9]  Projection Composite
     ├─ VirtualShadowMapCompositeTileVS/PS … ShadowMask → SceneColor のライティングへ合成
     └─ VirtualShadowMapCompositeFromMaskBitsPS … MaskBits から合成（One Pass Projection 時）
```

---

## 各ステップの詳細

### [1] Cache GPU Invalidation

| 項目 | 内容 |
|-----|------|
| **概要** | インスタンスの動的/静的状態変化（変形・移動）を検出し、該当 VSM ページを無効化 |
| **CPU 関数** | `FVirtualShadowMapCacheManager::ProcessInvalidations()` |
| **シェーダー** | `VirtualShadowMapCacheGPUInvalidation.usf` / `VirtualShadowMapCacheLoadBalancer.usf` |
| **出力** | `OutInvalidationQueue`（無効化ページリスト）, 更新済み `InOutViewInstanceState` |
| **CPU 詳細** | [[b_vsm_cache_manager]] |

### [2] Page Marking

| 項目 | 内容 |
|-----|------|
| **概要** | GBuffer・Hair・水面・Froxel を参照して、このフレームに必要な VSM ページにフラグを立てる |
| **CPU 関数** | `FVirtualShadowMapArray::BeginMarkPages()` |
| **シェーダー** | `VirtualShadowMapPageMarking.usf` |
| **出力** | `OutPageRequestFlags`（要求ページフラグビットマスク）|
| **CPU 詳細** | [[a_vsm_array]] |

### [3–4] Page Management + Physical Page Management

| 項目 | 内容 |
|-----|------|
| **概要** | 要求フラグを MIP 階層に伝播 → 物理ページを割り当て → 仮想ページテーブルを更新 |
| **CPU 関数** | `FVirtualShadowMapArray::BuildPageAllocations()` |
| **シェーダー** | `VirtualShadowMapPageManagement.usf` / `VirtualShadowMapPhysicalPageManagement.usf` |
| **出力** | `PageTable`（仮想→物理マッピング）, `PhysicalPagePool`（物理ページアトラス）|
| **CPU 詳細** | [[a_vsm_array]] |

### [5–6] Compact Views + Build Per-Page Draw Commands

| 項目 | 内容 |
|-----|------|
| **概要** | 描画が必要なビュー＋ページを絞り込み、インスタンスをカリングして Indirect Draw Arguments を生成 |
| **CPU 関数** | `FVirtualShadowMapArray::BuildPageAllocations()` 後半 / `RenderVirtualShadowMapsNonNanite()` |
| **シェーダー** | `VirtualShadowMapCompactViews.usf` / `VirtualShadowMapBuildPerPageDrawCommands.usf` |
| **出力** | Indirect Draw Arguments, InstanceId/PageInfo バッファ |
| **CPU 詳細** | [[a_vsm_array]] |

### [7] Shadow Depth Rendering

| 項目 | 内容 |
|-----|------|
| **概要** | [6] の Draw Commands を使って物理ページアトラスに深度を描画 |
| **CPU 関数** | `RenderVirtualShadowMapsNanite()` / `RenderVirtualShadowMapsNonNanite()` |
| **シェーダー** | Nanite: `NaniteRasterizer.usf`; 非 Nanite: `ShadowDepth.usf` |
| **出力** | `PhysicalPagePoolDepth`（物理ページアトラス上の深度）|
| **CPU 詳細** | [[a_vsm_array]] |

### [8] Projection（SMRT）

| 項目 | 内容 |
|-----|------|
| **概要** | 各ライトのシャドウ判定。SMRT（Shadow Map Ray Tracing）で半影推定・ソフトシャドウを生成 |
| **CPU 関数** | `RenderVirtualShadowMapProjection()` |
| **シェーダー** | `VirtualShadowMapProjection.usf#VirtualShadowMapProjection` |
| **出力** | `ShadowMaskTexture`（シャドウマスク float2: 影強度 + サブサーフェス）|
| **CPU 詳細** | [[d_vsm_projection]] |

### [9] Projection Composite

| 項目 | 内容 |
|-----|------|
| **概要** | ShadowMask をライティングパスの Shadow Attenuation に合成 |
| **CPU 関数** | `CompositeVirtualShadowMapMask()` |
| **シェーダー** | `VirtualShadowMapProjectionComposite.usf` |
| **出力** | Shadow Attenuation テクスチャ（SceneColor ライティングで参照）|
| **CPU 詳細** | [[d_vsm_projection]] |

---

## AsyncCompute との関係

VSM のページマーキングと物理ページ割り当ては GBuffer Pass の後に順次実行される。  
一部の処理（`ProcessInvalidations`）は前フレームのデータを参照するため、  
非同期実行するには GPU フェンスによる同期が必要。

---

## VSM シェーダー ↔ CPU 関数 対応表

| シェーダーファイル | 主要エントリポイント | CPU 関数 | グループ |
|----------------|----------------|---------|---------|
| `VirtualShadowMapCacheGPUInvalidation.usf` | `VSMUpdateViewInstanceStateCS` / `ProcessInvalidationQueueGPUCS` / `VSMResetInstanceStateCS` | `ProcessInvalidations()` | [[a_Invalidation]] |
| `VirtualShadowMapCacheLoadBalancer.usf` | `InvalidateInstancePagesLoadBalancerCS` | `ProcessInvalidations()` | [[a_Invalidation]] |
| `VirtualShadowMapPageMarking.usf` | `PruneLightGridCS` / `GeneratePageFlagsFromPixels` / `GeneratePageFlagsFromFroxelsCS` / `MarkCoarsePages` | `BeginMarkPages()` | [[b_PageManagement]] |
| `VirtualShadowMapPageManagement.usf` | `GenerateHierarchicalPageFlags` / `PropagateMappedMips` | `BuildPageAllocations()` | [[b_PageManagement]] |
| `VirtualShadowMapPhysicalPageManagement.usf` | `UpdatePhysicalPages` / `AllocateNewPageMappingsCS` / `AppendPhysicalPageLists` | `BuildPageAllocations()` | [[b_PageManagement]] |
| `VirtualShadowMapCompactViews.usf` | `CompactViewsVSM_CS` | `BuildPageAllocations()` | [[c_DrawCommands]] |
| `VirtualShadowMapBuildPerPageDrawCommands.usf` | `CullPerPageDrawCommandsCs` / `AllocateCommandInstanceOutputSpaceCs` / `OutputCommandInstanceListsCs` | `RenderVirtualShadowMapsNonNanite()` | [[c_DrawCommands]] |
| `VirtualShadowMapShadowCasterBounds.usf` | `MainVS` / `MainPS` | `RenderVirtualShadowMapsNonNanite()` | [[c_DrawCommands]] |
| `VirtualShadowMapProjection.usf` | `VirtualShadowMapProjection` | `RenderVirtualShadowMapProjection()` | [[d_Projection]] |
| `VirtualShadowMapProjectionComposite.usf` | `VirtualShadowMapCompositePS` / `VirtualShadowMapCompositeFromMaskBitsPS` | `CompositeVirtualShadowMapMask()` | [[d_Projection]] |
| `VirtualShadowMapThrottle.usf` | `ProcessPrevFramePerfDataCS` / `UpdateThrottleParametersCS` | `UpdateThrottleParameters()` | [[e_Misc]] |
| `VirtualShadowMapCopyStats.usf` | `CopyStatsCS` | `PostRender()` | [[e_Misc]] |
| `VirtualShadowMapPrintStats.usf` | `LogVirtualSmStatsCS` | `LogStats()` | [[e_Misc]] |
| `VirtualShadowMapDebug.usf` | `DebugVisualizeVirtualSmCS` | `AddDebugVisualizationPasses()` | [[e_Misc]] |
