# GPU Nanite ソースマップ

- 対象: Nanite GPU シェーダー（2パスカリング + ラスタライズ + Shade Binning + GBuffer Export）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_nanite_gpu_overview]]

PrimitiveFilter → Instance 階層 → Cluster Culling（Pass1 Main / Pass2 Post）→ Raster Binning → SW/HW ラスタライザ →
Shade Binning → GBuffer/Depth Export の流れ。`r.Nanite.AsyncRasterization=1` で GBuffer と並列実行可。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/Nanite/*.usf` |
| CPU | `Renderer/Private/Nanite/*.cpp` |

---

## ファイル → シェーダー対応

### Culling

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `NanitePrimitiveFilter.usf` | `PrimitiveFilter` | `AddPrimitiveFilterPass()` | [[a_Culling]] |
| `NaniteInstanceHierarchyCulling.usf` | `InstanceHierarchyCellChunkCull_CS` / `InstanceHierarchyChunkCull_CS` | `DispatchSetupPipeline()` | 同 |
| `NaniteClusterCulling.usf` | `NodeAndClusterCull`（Pass1/Pass2） | `IRenderer::DrawGeometry()` | 同 |

### Rasterizer

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `NaniteRasterBinning.usf` | `RasterBinBuild` / `RasterBinReserve` / `RasterBinFinalize` | `Nanite::RasterizePass()` | [[b_Rasterizer]] |
| `NaniteRasterizer.usf` | `MicropolyRasterize` / `HWRasterizeMS` / `HWRasterizePS` | 同上 | 同 |

### Shading

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `NaniteShadeBinning.usf` | `ShadingBinBuildCS` / `ShadingBinReserveCS` | `BuildShadingCommands()` | [[c_Shading]] |
| `NaniteExportGBuffer.usf` | `EmitSceneDepthPS` / `EmitSceneStencilPS` / `EmitHitProxyIdPS` 等 | `DispatchBasePass()` | 同 |

### Depth

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `NaniteDepthDecode.usf` | `DepthDecode` | `AddDepthDecodePass()` | [[d_Depth]] |
| `NaniteDepthExport.usf` | `DepthExport` | `AddDepthExportPass()` | 同 |

### 拡張

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `NaniteStreaming.usf` / `NaniteTranscode.usf` | `ClearStreamingRequestCount` / `TranscodePageToGPU` | `FStreamingManager::UpdateStreaming()` | [[e_Extensions]] |
| `NaniteRayTracing.usf` | `NaniteRayTracingStreamingTraversalCS` | `UpdateRayTracingScene()` | 同 |
| `NaniteStreamOut.usf` | `NaniteStreamOutTraversalCS` / `NaniteStreamOutCS` | `FStreamOutRequests::DispatchWork()` | 同 |
| `NaniteEmitShadow.usf` | `EmitShadowMapPS` / `EmitCubemapShadowVS/GS/PS` | `RenderShadowDepthMap()` | 同 |
| `NaniteScatterUpdates.usf` | — | GPUScene 更新 | 同 |

---

## GPU データフロー

```
[1]  NanitePrimitiveFilter.usf:PrimitiveFilter                          → RWPrimitiveFilterBuffer
[2]  NaniteInstanceHierarchyCulling.usf                                 → インスタンス候補
[3]  NaniteClusterCulling.usf:NodeAndClusterCull (CULLING_PASS_OCCLUSION_MAIN)
[4]  NaniteRasterBinning.usf:RasterBinBuild
[5]  NaniteRasterizer.usf:MicropolyRasterize (SW)
[6]  NaniteRasterizer.usf:HWRasterizeMS / HWRasterizePS (HW)            → VisBuffer64
[7]  HZB Build                                                           → HZBFurthest
[8]  NaniteClusterCulling.usf:NodeAndClusterCull (CULLING_PASS_OCCLUSION_POST)
[9-11] Raster Binning + Rasterizer Pass 2                                → VisBuffer64（追記）
[12] NaniteShadeBinning.usf:ShadingBinBuildCS → ShadingBinReserveCS     → ShadingBinData + Args
[13] NaniteExportGBuffer.usf:EmitSceneDepthPS / EmitSceneStencilPS 等   → GBuffer + ShadingMask
[14] NaniteDepthExport.usf:DepthExport                                   → SceneDepth + HTile
```

---

## AsyncCompute

```
Graphics Queue                  AsyncCompute Queue
────────────────────────────    ──────────────────────────────
BasePass（非 Nanite）
                             ←  Nanite Culling + Rasterizer
                                 （r.Nanite.AsyncRasterization=1 時）
Shade Binning
GBuffer Export
Depth Export
```

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.Nanite.AsyncRasterization` | `NaniteSceneRendering.cpp` |
| `r.Nanite.ProgrammableRaster` | 同 |
| `r.Nanite.Culling` | `NaniteCommandContext.cpp` |
| `r.Nanite.ShadeBinning` | `NaniteShading.cpp` |

---

## Details/Reference 一覧

| グループ | 詳細 |
|---------|------|
| a: Culling | [[detail_cull_raster]] |
| b: Rasterizer | [[detail_rasterizer]] |
| c: Shading | [[detail_materials_shading]] |
| d: Depth | [[detail_depth]] |
| e: Extensions | [[detail_ray_tracing]] / [[detail_tess_voxel]] |

---

## ue5-dive 起点

- 「Nanite エントリ」 → `DispatchSetupPipeline()` → `IRenderer::DrawGeometry()`
- 「Cluster Culling」 → `NaniteClusterCulling.usf:NodeAndClusterCull` + `CULLING_PASS_OCCLUSION_MAIN/POST`
- 「SW ラスタ」 → `NaniteRasterizer.usf:MicropolyRasterize`
- 「HW ラスタ」 → `NaniteRasterizer.usf:HWRasterizeMS/HWRasterizePS`
- 「Shade Binning」 → `NaniteShadeBinning.usf:ShadingBinBuildCS`
- 「GBuffer Export」 → `NaniteExportGBuffer.usf:EmitSceneDepthPS`
- 「Depth Export + HTile」 → `NaniteDepthExport.usf:DepthExport`
