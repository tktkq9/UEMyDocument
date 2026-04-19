# GPU VirtualShadowMaps ソースマップ

- 対象: VSM GPU シェーダー（Invalidation + PageMarking + PageManagement + Draw + SMRT Projection）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_vsm_gpu_overview]]

仮想ページテーブル + 物理ページアトラスで差分更新。Page Marking → 階層伝播 → 物理割当 →
Per-Page Draw Commands → Nanite/非 Nanite で深度描画 → SMRT で projection。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/VirtualShadowMaps/*.usf` |
| CPU | `Renderer/Private/VirtualShadowMaps/*.cpp` |

---

## ファイル → シェーダー対応

### Invalidation

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `VirtualShadowMapCacheGPUInvalidation.usf` | `VSMUpdateViewInstanceStateCS` / `ProcessInvalidationQueueGPUCS` / `VSMResetInstanceStateCS` | `FVirtualShadowMapCacheManager::ProcessInvalidations()` | [[a_Invalidation]] |
| `VirtualShadowMapCacheLoadBalancer.usf` | `InvalidateInstancePagesLoadBalancerCS` | 同上 | 同 |

### Page Management

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `VirtualShadowMapPageMarking.usf` | `PruneLightGridCS` / `GeneratePageFlagsFromPixels` / `GeneratePageFlagsFromFroxelsCS` / `MarkCoarsePages` | `FVirtualShadowMapArray::BeginMarkPages()` | [[b_PageManagement]] |
| `VirtualShadowMapPageManagement.usf` | `GenerateHierarchicalPageFlags` / `PropagateMappedMips` | `BuildPageAllocations()` | 同 |
| `VirtualShadowMapPhysicalPageManagement.usf` | `UpdatePhysicalPages` / `AllocateNewPageMappingsCS` / `AppendPhysicalPageLists` / `ClearPageTableCS` / `PackAvailablePages` / `InitPageRectBounds` / `UpdatePhysicalPageAddresses` | 同上 | 同 |

### Draw Commands

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `VirtualShadowMapCompactViews.usf` | `CompactViewsVSM_CS` | `BuildPageAllocations()` | [[c_DrawCommands]] |
| `VirtualShadowMapBuildPerPageDrawCommands.usf` | `CullPerPageDrawCommandsCs` / `AllocateCommandInstanceOutputSpaceCs` / `OutputCommandInstanceListsCs` / `ComputeExplicitChunkDrawsViewMask` | `RenderVirtualShadowMapsNonNanite()` | 同 |
| `VirtualShadowMapShadowCasterBounds.usf` | `MainVS` / `MainPS` | 同上 | 同 |

### Projection（SMRT）

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `VirtualShadowMapProjection.usf` | `VirtualShadowMapProjection` | `RenderVirtualShadowMapProjection()` | [[d_Projection]] |
| `VirtualShadowMapProjectionComposite.usf` | `VirtualShadowMapCompositePS` / `VirtualShadowMapCompositeFromMaskBitsPS` | `CompositeVirtualShadowMapMask()` | 同 |

### Misc / Debug

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `VirtualShadowMapThrottle.usf` | `ProcessPrevFramePerfDataCS` / `UpdateThrottleParametersCS` | `UpdateThrottleParameters()` | [[e_Misc]] |
| `VirtualShadowMapCopyStats.usf` | `CopyStatsCS` | `PostRender()` | 同 |
| `VirtualShadowMapPrintStats.usf` | `LogVirtualSmStatsCS` | `LogStats()` | 同 |
| `VirtualShadowMapDebug.usf` | `DebugVisualizeVirtualSmCS` | `AddDebugVisualizationPasses()` | 同 |

---

## GPU データフロー

```
[1] Cache Invalidation              FVirtualShadowMapCacheManager::ProcessInvalidations
    VirtualShadowMapCacheGPUInvalidation.usf + CacheLoadBalancer.usf

[2] Page Marking                    FVirtualShadowMapArray::BeginMarkPages
    VirtualShadowMapPageMarking.usf
      PruneLightGrid / GeneratePageFlagsFromPixels / GeneratePageFlagsFromFroxelsCS / MarkCoarsePages
    → OutPageRequestFlags

[3-4] Page Management                FVirtualShadowMapArray::BuildPageAllocations
    VirtualShadowMapPageManagement.usf:GenerateHierarchicalPageFlags / PropagateMappedMips
    VirtualShadowMapPhysicalPageManagement.usf:UpdatePhysicalPages / AllocateNewPageMappingsCS ...
    → PageTable + PhysicalPagePool

[5] Compact Views                    同上後半
    VirtualShadowMapCompactViews.usf:CompactViewsVSM_CS

[6] Build Per-Page Draw Commands    RenderVirtualShadowMapsNonNanite
    VirtualShadowMapBuildPerPageDrawCommands.usf:CullPerPageDrawCommandsCs ...
    → Indirect Draw Args + InstanceId/PageInfo バッファ

[7] Shadow Depth Rendering          RenderVirtualShadowMapsNanite / NonNanite
    Nanite 経路: NaniteRasterizer.usf
    非 Nanite : ShadowDepth.usf
    → PhysicalPagePoolDepth

[8] Projection（SMRT）              RenderVirtualShadowMapProjection
    VirtualShadowMapProjection.usf:VirtualShadowMapProjection
    → ShadowMaskTexture (float2: 影強度 + SSS)

[9] Composite                       CompositeVirtualShadowMapMask
    VirtualShadowMapProjectionComposite.usf:VirtualShadowMapCompositePS / FromMaskBitsPS
    → Shadow Attenuation → SceneColor ライティングで参照
```

---

## Details/Reference 一覧

| グループ | 詳細 |
|---------|------|
| a: Invalidation | [[detail_invalidation]] |
| b: PageManagement | [[detail_page_management]] |
| c: DrawCommands | [[detail_draw_commands]] |
| d: Projection | [[detail_projection]] |
| e: Misc | [[detail_misc]] |

---

## ue5-dive 起点

- 「ページマーキング」 → `VirtualShadowMapPageMarking.usf:GeneratePageFlagsFromPixels`
- 「物理ページ割当」 → `VirtualShadowMapPhysicalPageManagement.usf:UpdatePhysicalPages`
- 「Per-Page Draw」 → `VirtualShadowMapBuildPerPageDrawCommands.usf:CullPerPageDrawCommandsCs`
- 「SMRT Projection」 → `VirtualShadowMapProjection.usf:VirtualShadowMapProjection`
- 「One Pass Projection」 → `VirtualShadowMapProjectionComposite.usf:VirtualShadowMapCompositeFromMaskBitsPS`
