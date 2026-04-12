# Nanite GPU 処理概要

- グループ: Nanite GPU
- CPU 概要: [[03_nanite_overview]]
- CPU 詳細: [[a_nanite_cull_raster]] | [[b_nanite_materials_shading]] | [[c_nanite_visibility]] | [[d_nanite_ray_tracing]] | [[e_nanite_tess_voxel]]

---

## Nanite GPU パイプライン実行順

Nanite は **2パスカリング + ラスタライズ** + **遅延マテリアルシェーディング** を核とする GPU Driven レンダリングシステム。

```
[1]  PrimitiveFilter
[2]  Instance Hierarchy Culling
[3]  Cluster Culling Pass 1（Main — 前フレームHZBで粗いオクルージョン）
[4]  Raster Binning Pass 1
[5]  SW Rasterizer Pass 1（MicropolyRasterize — 小三角形 Compute）
[6]  HW Rasterizer Pass 1（HWRasterizeMS/PS — 大三角形 Mesh Shader/VS+PS）
       ↓ VisBuffer64 書き込み済み
[7]  HZB Build（VisBuffer 深度から再構築）
[8]  Cluster Culling Pass 2（Post — 新HZBで再テスト）
[9]  Raster Binning Pass 2
[10] SW Rasterizer Pass 2
[11] HW Rasterizer Pass 2
       ↓ VisBuffer64 最終確定
[12] Shade Binning（VisBuffer ピクセルをマテリアルビン別に分類）
[13] GBuffer Export / Material Shading（ビン単位で材質評価）
[14] Depth Export（VisBuffer → ハードウェア深度バッファ + HTile）
```

---

## 各ステップの詳細

### [1] PrimitiveFilter

| 項目 | 内容 |
|-----|------|
| **概要** | HiddenPrimitivesList / ShowOnlyPrimitivesList をチェックして不可視プリミティブをフラグ立て |
| **CPU 関数** | `FNaniteCommandContext::DispatchSetupPipeline()` → `AddPrimitiveFilterPass()` |
| **シェーダー** | `NanitePrimitiveFilter.usf#PrimitiveFilter` |
| **出力** | `RWPrimitiveFilterBuffer`（uint per primitive） |
| **CPU 詳細** | [[a_nanite_cull_raster]] |

### [2] Instance Hierarchy Culling

| 項目 | 内容 |
|-----|------|
| **概要** | 2レベルのインスタンス階層（Cell Chunk → Chunk）で粗いフラスタムカリング |
| **CPU 関数** | `FNaniteCommandContext::DispatchSetupPipeline()` |
| **シェーダー** | `NaniteInstanceHierarchyCulling.usf#InstanceHierarchyCellChunkCull_CS` / `InstanceHierarchyChunkCull_CS` |
| **出力** | インスタンス候補バッファ（→ Cluster Culling の入力） |
| **CPU 詳細** | [[a_nanite_cull_raster]] |

### [3] Cluster Culling Pass 1（Main）

| 項目 | 内容 |
|-----|------|
| **概要** | BVH 階層トラバーサル。フラスタム + LOD 選択 + 前フレーム HZB オクルージョン |
| **CPU 関数** | `FInstanceHierarchyContext::RasterizeGeometry()` 内 `NodeAndClusterCull` ディスパッチ |
| **シェーダー** | `NaniteClusterCulling.usf#NodeAndClusterCull`（`CULLING_PASS=CULLING_PASS_OCCLUSION_MAIN`）|
| **出力** | `RWQueueState`（可視クラスタリスト）/ `RWOccludedInstances`（オクルード候補）|
| **CPU 詳細** | [[a_nanite_cull_raster]] |

### [4–6] Raster Binning + Rasterizer Pass 1

| 項目 | 内容 |
|-----|------|
| **概要** | 可視クラスタを SW/HW ビンに仕分け → Compute（SW）+ Mesh Shader/VS+PS（HW）でラスタライズ |
| **CPU 関数** | `Nanite::RasterizePass()` |
| **シェーダー** | `NaniteRasterBinning.usf#RasterBinBuild` → `NaniteRasterizer.usf#MicropolyRasterize` / `HWRasterizeMS` / `HWRasterizePS` |
| **出力** | `VisBuffer64`（depth+clusterID+triIdx）, `DbgBuffer64/32` |
| **CPU 詳細** | [[a_nanite_cull_raster]] |

### [7] HZB Build

| 項目 | 内容 |
|-----|------|
| **概要** | Pass 1 の VisBuffer から Hierarchical Z Buffer を再生成（Pass 2 のオクルージョン判定用）|
| **CPU 関数** | `BuildHZBFurthest()` / `BuildHZBNearest()` |
| **シェーダー** | `HZBBuild.usf`（Nanite 専用ではないが Nanite が呼び出す）|
| **出力** | `HZBFurthest` テクスチャ（→ Pass 2 Cluster Culling の入力）|

### [8–11] Cluster Culling Pass 2 + Rasterizer Pass 2

| 項目 | 内容 |
|-----|------|
| **概要** | Pass 1 でオクルードされたクラスタを新HZBで再テスト → 可視なら再ラスタライズ |
| **CPU 関数** | 同 `Nanite::RasterizePass()` 内の Post パス |
| **シェーダー** | `NaniteClusterCulling.usf#NodeAndClusterCull`（`CULLING_PASS=CULLING_PASS_OCCLUSION_POST`）→ 同ラスタライザ |
| **出力** | `VisBuffer64`（追記）|

### [12] Shade Binning

| 項目 | 内容 |
|-----|------|
| **概要** | VisBuffer の各ピクセルのマテリアルを判定し、シェーディングビン別にソート・集約 |
| **CPU 関数** | `Nanite::BuildShadingCommands()` |
| **シェーダー** | `NaniteShadeBinning.usf#ShadingBinBuildCS` → `ShadingBinReserveCS` |
| **出力** | `ShadingBinData`（ピクセルリスト）, `ShadingBinArgs`（Indirect Dispatch 引数）|
| **CPU 詳細** | [[b_nanite_materials_shading]] |

### [13] GBuffer Export / Material Shading

| 項目 | 内容 |
|-----|------|
| **概要** | シェーディングビン単位で Indirect Draw/Dispatch。VisBuffer を参照してマテリアル評価 → GBuffer 書き込み |
| **CPU 関数** | `Nanite::DispatchBasePass()` |
| **シェーダー** | `NaniteExportGBuffer.usf#EmitSceneDepthPS` / `EmitSceneStencilPS` / `EmitHitProxyIdPS` 等 |
| **出力** | GBuffer A/B/C/D, SceneColor（エミッシブ）, ShadingMask |
| **CPU 詳細** | [[b_nanite_materials_shading]] |

### [14] Depth Export

| 項目 | 内容 |
|-----|------|
| **概要** | VisBuffer64 の深度値を ハードウェア SceneDepth バッファと HTile に書き込む |
| **CPU 関数** | `Nanite::AddDepthExportPass()` |
| **シェーダー** | `NaniteDepthExport.usf#DepthExport` |
| **出力** | `SceneDepth`（ハードウェア深度）, `HTile`（深度圧縮メタデータ）, `ShadingMask` |
| **CPU 詳細** | [[a_nanite_cull_raster]] |

---

## AsyncCompute との関係

```
Graphics Queue                  AsyncCompute Queue
─────────────────────────────   ──────────────────────────────
GBuffer Pass（非Nanite部分）
                              ←  Nanite Culling + Rasterizer
                                  （r.Nanite.AsyncRasterization=1 時）
Shade Binning
GBuffer Export
Depth Export
```

Lumen とは異なり、Nanite のカリング/ラスタライズは AsyncCompute で走らせることで  
従来の GBuffer パスと並列実行できる（`r.Nanite.AsyncRasterization`）。

---

## 拡張パス（メインパイプライン外）

| パス | シェーダー | 概要 |
|-----|----------|------|
| Streaming / Transcode | `NaniteStreaming.usf` / `NaniteTranscode.usf` | GPU フィードバック収集 → ページデコード → Atlas 転送 |
| Ray Tracing | `NaniteRayTracing.usf` | BLAS 構築用 BVH トラバーサル（ストリーミングパス）|
| Stream Out | `NaniteStreamOut.usf` | Nanite クラスタ → CPU 可読 Mesh への展開 |
| Depth Decode | `NaniteDepthDecode.usf` | デバッグ/VSM 用に HTile から深度値をデコード |
| Emit Shadow | `NaniteEmitShadow.usf` | Nanite メッシュのシャドウマップ出力 |
| Scatter Updates | `NaniteScatterUpdates.usf` | GPU Scene 更新データの Scatter 書き込み |
| Primitive Filter | `NanitePrimitiveFilter.usf` | エディタ・ゲームフィルタリング |

---

## Nanite シェーダー ↔ CPU 関数 対応表

| シェーダーファイル | 主要エントリポイント | CPU 関数 | グループ |
|----------------|----------------|---------|---------|
| `NanitePrimitiveFilter.usf` | `PrimitiveFilter` | `AddPrimitiveFilterPass()` | [[a_Culling]] |
| `NaniteInstanceHierarchyCulling.usf` | `InstanceHierarchyCellChunkCull_CS` / `InstanceHierarchyChunkCull_CS` | `DispatchSetupPipeline()` | [[a_Culling]] |
| `NaniteClusterCulling.usf` | `NodeAndClusterCull`（Pass1/2）| `IRenderer::DrawGeometry()` | [[a_Culling]] |
| `NaniteRasterBinning.usf` | `RasterBinBuild` / `RasterBinReserve` / `RasterBinFinalize` | `Nanite::RasterizePass()` | [[b_Rasterizer]] |
| `NaniteRasterizer.usf` | `MicropolyRasterize` / `HWRasterizeMS` / `HWRasterizePS` | `Nanite::RasterizePass()` | [[b_Rasterizer]] |
| `NaniteShadeBinning.usf` | `ShadingBinBuildCS` / `ShadingBinReserveCS` | `BuildShadingCommands()` | [[c_Shading]] |
| `NaniteExportGBuffer.usf` | `EmitSceneDepthPS` / `EmitSceneStencilPS` 等 | `DispatchBasePass()` | [[c_Shading]] |
| `NaniteDepthDecode.usf` | `DepthDecode` | `AddDepthDecodePass()` | [[d_Depth]] |
| `NaniteDepthExport.usf` | `DepthExport` | `AddDepthExportPass()` | [[d_Depth]] |
| `NaniteStreaming.usf` / `NaniteTranscode.usf` | `ClearStreamingRequestCount` / `TranscodePageToGPU` | `FStreamingManager::UpdateStreaming()` | [[e_Extensions]] |
| `NaniteRayTracing.usf` | `NaniteRayTracingStreamingTraversalCS` | `UpdateRayTracingScene()` | [[e_Extensions]] |
| `NaniteStreamOut.usf` | `NaniteStreamOutTraversalCS` / `NaniteStreamOutCS` | `FStreamOutRequests::DispatchWork()` | [[e_Extensions]] |
| `NaniteEmitShadow.usf` | `EmitShadowMapPS` / `EmitCubemapShadowVS/GS/PS` | `RenderShadowDepthMap()` | [[e_Extensions]] |
