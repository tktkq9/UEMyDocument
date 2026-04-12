# GPU 処理概要（UE5 Deferred Rendering）

- 取得日: 2026-04-12
- 対象: `DeferredShadingRenderer.cpp` を中心とした GPU 実行フロー
- 上位: [[01_rendering_overview]]
- 関連 CPU リファレンス: [[12_rhi_overview]] | [[10_rdg_overview]]

---

## GPU 処理とは

UE5 の描画は **CPU（Render Thread / RHI Thread）がコマンドを積み**、**GPU が非同期で実行する** という2段構造。  
「GPU 処理の順序」とは RDG が確定した **Pass 依存グラフのトポロジカル順** であり、  
`FRDGBuilder::Execute()` が実際の `ID3D12GraphicsCommandList` コマンドに変換して GPU へ投入する。

本文書では **Deferred Shading + Lumen 有効** の典型的なフレームで GPU が実行する処理を、  
**実行順 → CPU 担当関数 → シェーダー / エントリポイント** の対応で一覧する。

---

## フレーム全体の GPU 実行順

```
─── フレーム開始 ────────────────────────────────────────────────

[1]  GPU Scene Update
[2]  Shadow Depth Rendering
[3]  Depth Pre-pass（Z Pre-pass）
[4]  Nanite Rasterization
[5]  Lumen Surface Cache Update
     [5a] Card Capture
     [5b] Surface Cache Lighting（Direct + Radiosity）
[6]  Base Pass（GBuffer 書き込み）
[7]  ── AsyncCompute 並列区間 START ──────────
     [7a] Lumen Screen Probe Gather（Diffuse GI）
     [7b] Lumen Reflections
     ── AsyncCompute 並列区間 END ────────────
[8]  HZB（Hierarchical Z-Buffer）構築
[9]  Occlusion Queries
[10] Screen Space AO（Lumen GI 有効時は Lumen AO のみ）
[11] Direct Lighting（Deferred Lighting Pass）
     [11a] Clustered / Tiled Deferred
     [11b] Shadow Projection
     [11c] Virtual Shadow Map Projection
[12] Lumen Final Composite（Diffuse GI + 反射 → SceneColor）
[13] Sky / Atmosphere
[14] Fog（Height Fog / Volumetric Fog）
[15] Translucency（OIT / Standard）
[16] Post-processing Chain
     [16a] Temporal AA (TSR/TAA)
     [16b] Bloom / Lens Flare
     [16c] Tone Mapping
     [16d] UI Composite

─── Present ─────────────────────────────────────────────────────
```

---

## 詳細テーブル

### [1] GPU Scene Update

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | プリミティブ・トランスフォーム・ライトデータを GPU に転送 |
| **CPU 関数** | `GPUScene::UploadDynamicPrimitiveShaderDataForView()` (`DeferredShadingRenderer.cpp:2172`) |
| **シェーダー** | Compute（`UpdatePrimitivesToRender.usf`） |
| **出力** | `PrimitiveSceneDataTexture`（GPU Scene Buffer） |

---

### [2] Shadow Depth Rendering

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | 各ライトのシャドウマップ / VSM ページへの深度書き込み |
| **CPU 関数** | `RenderShadowDepthMaps()` → `VirtualShadowMapArray.RenderShadowMaps()` |
| **シェーダー** | VS/PS: `ShadowDepth.usf`（`Main()` / `MainPS()`）; Nanite: `NaniteShadow.usf` |
| **出力** | Shadow Depth テクスチャ / VSM Physical Page Pool |
| **CPU 詳細** | [[12_vsm_overview]] / VSM Reference |

---

### [3] Depth Pre-pass（Z Pre-pass）

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | 不透明オブジェクトを先行描画して SceneDepth を確定 |
| **CPU 関数** | `RenderPrePass()` (`DepthRendering.cpp`) |
| **シェーダー** | VS: `PositionOnlyDepthVertexShader.usf`; PS: 省略可（Depth Only）|
| **出力** | `SceneDepthZ`（SceneTextures.Depth） |

---

### [4] Nanite Rasterization

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | HW + SW ラスタライザで Nanite メッシュを GBuffer に描画 |
| **CPU 関数** | `RenderNanite()` → `Nanite::RasterizePass()` |
| **シェーダー** | `NaniteRasterization.usf`（Compute）/ `NaniteEmitGBuffer.usf`（CS）|
| **出力** | GBuffer A/B/C/D, SceneDepth（Nanite 部分） |
| **CPU 詳細** | Nanite Reference（未作成） |

---

### [5a] Lumen Card Capture

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | Lumen Card（シーンの簡略表現）をテクスチャアトラスに描画 |
| **CPU 関数** | `UpdateLumenScene()` → `LumenScene::RenderCardCaptureViews()` (`LumenSceneRendering.cpp`) |
| **シェーダー** | VS: `LumenCardVertexShader.usf#Main()`; PS: `LumenCardPixelShader.usf#Main()`; CS: `LumenCardComputeShader.usf#Main()` |
| **出力** | Albedo Atlas / Normal Atlas / Emissive Atlas（Surface Cache テクスチャ群） |
| **GPU シェーダー詳細** | [[detail_card_capture]] / [[ref_card_shaders]] |

---

### [5b] Lumen Surface Cache Lighting

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | Surface Cache の各 Card に DirectLighting + Radiosity を焼き込み |
| **CPU 関数** | `RenderLumenSceneLighting()` (`LumenSceneLighting.cpp:217`) |
| **シェーダー** | `LumenSceneDirectLighting.usf` / `LumenSceneLighting.usf#CombineLumenSceneLightingCS()` / `Radiosity/LumenRadiosity.usf` |
| **出力** | DirectLighting Atlas / IndirectLighting Atlas → **FinalLighting Atlas** |
| **GPU シェーダー詳細** | [[detail_direct_lighting]] / [[ref_direct_lighting]] / [[detail_radiosity]] / [[ref_radiosity]] |

---

### [6] Base Pass（GBuffer）

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | 不透明マテリアルを GBuffer（A/B/C/D + Depth + Velocity）に書き込む |
| **CPU 関数** | `RenderBasePass()` (`BasePassRendering.cpp`) |
| **シェーダー** | VS: `BasePassVertexShader.usf`; PS: `BasePassPixelShader.usf`（マテリアル別生成） |
| **出力** | GBuffer A（BaseColor+Metallic）/ B（Normal+Roughness）/ C（Subsurface）/ D（CustomData） |
| **CPU 詳細** | BasePass Reference（未作成） |

---

### [7a] Lumen Screen Probe Gather（AsyncCompute）

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | Screen Space Probe でレイをトレースし Diffuse GI テクスチャを生成 |
| **CPU 関数** | `DispatchAsyncLumenIndirectLightingWork()` → `RenderLumenFinalGather()` (`IndirectLightRendering.cpp:889`) |
| **シェーダー** | `LumenScreenProbeTracing.usf` / `LumenScreenProbeGather.usf` / `LumenScreenProbeGatherTemporal.usf` / `LumenScreenProbeFiltering.usf` |
| **キュー** | AsyncCompute（DirectキューとGBuffer描画と並列）|
| **出力** | `FSSDSignalTextures::Textures[0-2]`（Diffuse GI バッファ群） |
| **GPU シェーダー詳細** | [[detail_screen_probe_gather]] / [[ref_screen_probe_gather]] |

---

### [7b] Lumen Reflections（AsyncCompute）

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | Roughness 別に反射レイをトレースし Resolve・テンポラル蓄積 |
| **CPU 関数** | `DispatchAsyncLumenIndirectLightingWork()` → `RenderLumenReflections()` |
| **シェーダー** | `LumenReflectionTracing.usf` / `LumenReflections.usf` / `LumenReflectionResolve.usf` / Denoiser 系 |
| **キュー** | AsyncCompute（ReflectionsMethod == Lumen かつ AsyncCompute 有効時）|
| **出力** | `FSSDSignalTextures::Textures[3]`（反射バッファ）|
| **GPU シェーダー詳細** | [[detail_reflections]] / [[ref_reflection_tracing]] |

---

### [8] HZB 構築

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | SceneDepth から Hierarchical Z-Buffer（ミップチェーン）を構築 |
| **CPU 関数** | `BuildHZB()` (`SceneHZB.cpp`) |
| **シェーダー** | `HZBOcclusion.usf`（Compute） |
| **出力** | `HZB`（遮蔽カリング・Screen Probe トレースに使用） |

---

### [9] Occlusion Queries

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | 前フレームの遮蔽クエリ結果を読み取り、オクルージョンカリングに使用 |
| **CPU 関数** | `RenderOcclusion()` |
| **シェーダー** | `OcclusionQueryVertexShader.usf`（VS のみ） |

---

### [10] Screen Space AO

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | SSAO または Lumen Short Range AO の計算 |
| **CPU 関数** | `CompositionLighting.ProcessAfterOcclusion()` → `AddAmbientOcclusionPasses()` |
| **シェーダー** | `ScreenSpaceAO.usf` / `LumenScreenSpaceBentNormal.usf`（Lumen 時）|
| **出力** | AO テクスチャ（`SceneTextures.ScreenSpaceAO`） |
| **GPU シェーダー詳細** | [[ref_restir_bent_normal]]（Lumen Bent Normal 部分）|

---

### [11] Direct Lighting（Deferred Lighting Pass）

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | GBuffer を読み取り、全ライトの直接照明を SceneColor に書き込む |
| **CPU 関数** | `RenderDeferredLighting()` (`DeferredShadingRenderer.cpp:3256`) |
| **シェーダー** | `DeferredLightPixelShaders.usf`（ライト種別 × 影あり/なし × Lumen等）|
| **出力** | `SceneColor`（直接照明を加算） |

---

### [12] Lumen Final Composite

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | Diffuse GI + 反射 + AO を GBuffer と合成して SceneColor に加算書き込み |
| **CPU 関数** | `RenderDiffuseIndirectAndAmbientOcclusion()` (`IndirectLightRendering.cpp:977`) |
| **シェーダー** | `DiffuseIndirectComposite.usf#MainPS()`（`FDiffuseIndirectCompositePS`） |
| **出力** | `SceneColor`（最終的な GI + 反射込みライティング） |
| **CPU 詳細** | [[g_lumen_final_composite]] |
| **GPU シェーダー詳細** | [[detail_diffuse_indirect_composite]] / [[ref_diffuse_indirect_composite]] |

---

### [13] Sky / Atmosphere

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | Physically Based Sky / Sky Atmosphere / Sky Light をレンダリング |
| **CPU 関数** | `RenderSkyAtmosphere()` / `RenderSkyAtmosphereMainView()` |
| **シェーダー** | `SkyAtmosphere.usf`（多数のLUTパス）/ `SkyPassVertexShader.usf` |
| **出力** | `SceneColor`（空・大気の寄与を加算） |

---

### [14] Fog

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | Height Fog / Volumetric Fog を SceneColor に合成 |
| **CPU 関数** | `RenderFog()` / `RenderVolumetricFog()` |
| **シェーダー** | `HeightFogPixelShader.usf` / `VolumetricFog.usf` |
| **出力** | `SceneColor`（霧の寄与を加算） |

---

### [15] Translucency

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | 半透明オブジェクトを深度順に描画 |
| **CPU 関数** | `RenderTranslucency()` (`TranslucentRendering.cpp`) |
| **シェーダー** | マテリアル別生成（BasePass 系 + `TranslucentLightingShaders.usf`） |
| **出力** | `SceneColor`（透明オブジェクトを上書き合成） |

---

### [16] Post-processing

| 項目 | 内容 |
|-----|------|
| **GPU 処理** | TAA/TSR → Bloom → Lens Flare → Tone Mapping → DOF → UI Composite |
| **CPU 関数** | `AddPostProcessingPasses()` (`PostProcessing.cpp`) |
| **シェーダー** | `TemporalAA.usf` / `TSR*.usf` / `Bloom.usf` / `TonemapCommon.usf` など |
| **出力** | 最終 `SceneColor`（SwapChain 表示用） |
| **CPU 詳細** | [[10_pp_overview]] → PostProcess 各 Reference |

---

## AsyncCompute 並列実行の詳細

```
Graphics キュー:
  [3] Depth Pre-pass
  [4] Nanite Rasterization
  [5] Lumen Card Capture & Lighting ← GPU 負荷の谷間
  [6] Base Pass (GBuffer)            ← 重い

AsyncCompute キュー（r.Lumen.AsyncCompute=1 時）:
  ─── GBuffer と並列 ──────────────────
  [7a] Screen Probe Gather
  [7b] Reflections（ReflectionsMethod==Lumen 時）
  ─────────────────────────────────────

  ※ GBuffer 完了後に FD3D12Queue::WaitForOtherQueue() でフェンス同期
  ※ AsyncCompute が終わらなかったステップは [12] で同期的に補完
```

---

## シェーダー別 CPU 対応一覧（Lumen）

| シェーダーファイル | エントリポイント | CPU 関数 | グループ |
|-----------------|---------------|---------|---------|
| `LumenCardVertexShader.usf` | `Main()` | `LumenScene::RenderCardCaptureViews()` | a |
| `LumenCardPixelShader.usf` | `Main()` | `LumenScene::RenderCardCaptureViews()` | a |
| `LumenCardComputeShader.usf` | `Main()` | `LumenScene::RenderCardCaptureViews()` | a |
| `SurfaceCache/LumenSurfaceCache.usf` | 複数 CS | `UpdateLumenScene()` | a |
| `LumenSceneDirectLighting.usf` | 複数 CS | `RenderDirectLightingForLumenScene()` | b |
| `LumenSceneLighting.usf` | `CombineLumenSceneLightingCS()` | `RenderLumenSceneLighting()` | b |
| `Radiosity/LumenRadiosity.usf` | 複数 CS | `RenderRadiosityForLumenScene()` | b |
| `LumenMeshSDFCulling.usf` | `CullMeshObjectsToProbesCS()` など | 各トレース関数 | c |
| `LumenScene.usf` | `LumenCardUpdateCS()` など | `UpdateLumenScene()` | c |
| `LumenRadianceCache.usf` | `InterpolateRadianceCacheCS()` | `LumenRadianceCache::Render()` | d |
| `LumenRadianceCacheUpdate.usf` | `MarkRadianceCacheProbesCS()` など | `LumenRadianceCache::Update()` | d |
| `LumenScreenProbeTracing.usf` | `LumenScreenProbeTraceCS()` | `RenderLumenScreenProbeGather()` | e |
| `LumenScreenProbeGather.usf` | `LumenScreenProbeGatherCS()` | `RenderLumenScreenProbeGather()` | e |
| `LumenScreenProbeGatherTemporal.usf` | `TemporallyAccumulateScreenProbesCS()` | `RenderLumenScreenProbeGather()` | e |
| `LumenScreenProbeFiltering.usf` | `FilterScreenProbesCS()` | `RenderLumenScreenProbeGather()` | e |
| `LumenReSTIRGather.usf` | `ReSTIRGatherCS()` など | `RenderLumenReSTIRGather()` | e |
| `LumenScreenSpaceBentNormal.usf` | `LumenScreenSpaceBentNormalCS()` | `RenderLumenScreenSpaceBentNormal()` | e |
| `LumenReflectionTracing.usf` | 複数 CS | `RenderLumenReflections()` | f |
| `LumenReflections.usf` | 複数 CS | `RenderLumenReflections()` | f |
| `LumenReflectionResolve.usf` | `ResolveReflectionsCS()` | `RenderLumenReflections()` | f |
| `LumenReflectionDenoiser*.usf` | 各 CS | `RenderLumenReflections()` | f |
| `DiffuseIndirectComposite.usf` | `MainPS()` | `RenderDiffuseIndirectAndAmbientOcclusion()` | g |
