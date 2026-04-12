# Nanite GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_nanite_gpu_overview.md` … Nanite GPU 全処理実行順 + CPU対応表

---

## Nanite GPU シェーダー（グループ別）

### a: Culling（2本）
- [x] `a_Culling/detail_culling.md`  … PrimitiveFilter + Instance Hierarchy + Cluster Culling 2パス
- [x] `a_Culling/ref_culling.md`     … NanitePrimitiveFilter / NaniteInstanceHierarchyCulling / NaniteClusterCulling エントリポイント

### b: Rasterizer（2本）
- [x] `b_Rasterizer/detail_rasterizer.md`  … Raster Binning + SW/HW ラスタライザ（2パス）
- [x] `b_Rasterizer/ref_rasterizer.md`     … NaniteRasterBinning / NaniteRasterizer エントリポイント

### c: Shading（2本）
- [x] `c_Shading/detail_shading.md`  … Shade Binning + GBuffer Export / マテリアルシェーディング
- [x] `c_Shading/ref_shading.md`     … NaniteShadeBinning / NaniteExportGBuffer エントリポイント

### d: Depth（2本）
- [x] `d_Depth/detail_depth.md`  … Depth Decode + Depth Export（VisBuffer → ハードウェア深度）
- [x] `d_Depth/ref_depth.md`     … NaniteDepthDecode / NaniteDepthExport エントリポイント

### e: Extensions（2本）
- [x] `e_Extensions/detail_extensions.md`  … Streaming / Transcode / Ray Tracing / Stream Out / Emit Shadow
- [x] `e_Extensions/ref_extensions.md`     … NaniteStreaming / NaniteTranscode / NaniteRayTracing / NaniteStreamOut / NaniteEmitShadow エントリポイント

---

合計: 概要 1 + Nanite シェーダー 10 = **11 ファイル**
