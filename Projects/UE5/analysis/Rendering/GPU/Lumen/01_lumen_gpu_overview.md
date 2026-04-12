# Lumen GPU 処理概要

- グループ: Lumen GPU
- CPU 概要: [[02_lumen_overview]]
- CPU 詳細: [[a_lumen_surface_cache]] | [[b_lumen_scene_lighting]] | [[c_lumen_tracing]] | [[d_lumen_radiance_cache]] | [[e_lumen_diffuse_gi]] | [[f_lumen_reflections]] | [[g_lumen_final_composite]]

---

## Lumen GPU パイプライン実行順

Lumen は **Surface Cache（オフスクリーンのライティングキャッシュ）** と  
**Screen Probe Gather（スクリーンスペース GI）** を組み合わせた動的グローバルイルミネーションシステム。

```
[Surface Cache 構築]
[1]  Card Capture
     ├─ LumenCardVertexShader / LumenCardPixelShader … 通常メッシュの Albedo/Normal/Emissive を Card アトラスに描画
     └─ LumenCardComputeShader … Nanite メッシュの Card シェーディング（Compute）

[Surface Cache ライティング]
[2]  Direct Lighting（Surface Cache）
     ├─ LumenSceneDirectLightingCullingCS … Card ごとにライトをカリング → LightTilesPerCardTile 生成
     ├─ LumenSceneDirectLightingShadowMaskCS … シャドウマスク計算（SW RT または VSM）
     ├─ LumenCardBatchDirectLightingCS … Irradiance を DirectLighting Atlas に書き込み
     └─ CombineLumenSceneLightingCS … DirectLighting + Radiosity → FinalLighting Atlas 合成

[3]  Radiosity（表面間接光）
     ├─ LumenRadiosityMarkUsedProbesCS … 使用プローブをマーク
     ├─ LumenRadiosityTraceProbesCS … プローブからレイをトレース（SDF/HW RT）
     ├─ LumenRadiosityFilterProbeRadianceCS … Probe Radiance をフィルタリング
     └─ LumenRadiosityIntegrateCS … Irradiance を Surface Cache に積分

[SDF トレーシング共有基盤]
[4]  Mesh SDF Culling + Tracing
     ├─ CullMeshSDFObjectsForViewCS … View フラスタム＋距離カリング
     └─ CompactCulledObjectsCS … 有効エントリをコンパクト化
     ※ トレース本体は各グループが LumenSoftwareRayTracing.ush を include して実行

[Radiance Cache 更新]
[5]  Radiance Cache Update
     ├─ AllocateUsedProbesCS … 使用中プローブを確定
     ├─ AllocateProbeTracesCS … 更新バジェットを割り当て
     ├─ TraceFromProbesCS … SDF / HW RT でプローブからレイをトレース
     ├─ FilterProbeRadianceWithGatherCS … フィルタリング
     └─ CalculateProbeIrradianceCS + FixupBordersAndGenerateMipsCS … Irradiance 変換 + ミップ生成

[Diffuse GI（AsyncCompute）]
[6]  Screen Probe Gather
     ├─ LumenScreenProbeTraceCS … Screen Probe からレイをトレース（SDF/HW RT/Screen Trace）
     ├─ LumenScreenProbeGatherCS … 近傍プローブから放射照度を収集
     ├─ TemporallyAccumulateScreenProbesCS … テンポラル蓄積
     └─ FilterScreenProbesCS … 空間フィルタリング → Diffuse GI バッファ

[7]  ReSTIR GI + Bent Normal（オプション）
     ├─ ReSTIRGatherCS … ReSTIR GI サンプリング（r.Lumen.ReSTIR 有効時）
     └─ LumenScreenSpaceBentNormalCS … Bent Normal 計算（短距離 AO 代替）

[反射（AsyncCompute）]
[8]  Reflection Tracing + Resolve + Denoise
     ├─ ReflectionTileClassificationBuildListsCS … Roughness でタイル分類
     ├─ ReflectionGenerateRaysCS … レイ方向・MIS 重みを計算
     ├─ ReflectionTraceScreenTexturesCS … スクリーントレース
     ├─ ReflectionTraceMeshSDFsCS + ReflectionTraceVoxelsCS … SDF トレース
     ├─ LumenReflectionResolveCS … Resolve（ダウンサンプル結果を再構成）
     ├─ LumenReflectionDenoiserTemporalCS … テンポラルデノイズ
     └─ LumenReflectionDenoiserSpatialCS … 空間デノイズ → 反射バッファ

[最終合成]
[9]  Final Composite（DiffuseIndirectComposite）
     └─ MainPS … Diffuse GI + 反射 + AO → SceneColor に加算合成
```

---

## 各ステップの詳細

### [1] Card Capture

| 項目 | 内容 |
|-----|------|
| **概要** | シーン内オブジェクトを Card（平面簡略表現）としてテクスチャアトラスに描画。WPO 無効化、Albedo/Normal/Emissive のみ書き込む |
| **CPU 関数** | `LumenScene::RenderCardCaptureViews()` (`LumenSceneRendering.cpp`) |
| **シェーダー** | VS: `LumenCardVertexShader.usf#Main()`; PS: `LumenCardPixelShader.usf#Main()`; CS: `LumenCardComputeShader.usf#Main()` |
| **出力** | Albedo Atlas / Normal Atlas / Emissive Atlas |
| **CPU 詳細** | [[a_lumen_surface_cache]] |
| **GPU シェーダー詳細** | [[detail_card_capture]] / [[ref_card_shaders]] |

---

### [2] Direct Lighting（Surface Cache）

| 項目 | 内容 |
|-----|------|
| **概要** | Surface Cache の各 Card に直接光を書き込む。ライトカリング → シャドウマスク → Irradiance 計算の3段階 |
| **CPU 関数** | `RenderDirectLightingForLumenScene()` (`LumenSceneDirectLighting.cpp`) |
| **シェーダー** | `LumenSceneDirectLightingCulling.usf` / `LumenSceneDirectLightingShadowMask.usf` / `LumenSceneDirectLighting.usf` |
| **出力** | DirectLighting Atlas |
| **CPU 詳細** | [[b_lumen_scene_lighting]] |
| **GPU シェーダー詳細** | [[detail_direct_lighting]] / [[ref_direct_lighting]] |

---

### [3] Radiosity（表面間接光）

| 項目 | 内容 |
|-----|------|
| **概要** | Surface Cache 上のプローブからレイをトレースして1バウンスの間接光を計算。FinalLighting Atlas に合算 |
| **CPU 関数** | `RenderRadiosityForLumenScene()` (`LumenRadiosity.cpp`) |
| **シェーダー** | `Radiosity/LumenRadiosity.usf`（複数 CS）, `LumenSceneLighting.usf#CombineLumenSceneLightingCS` |
| **出力** | IndirectLighting Atlas → FinalLighting Atlas（Direct + Indirect 合算） |
| **CPU 詳細** | [[b_lumen_scene_lighting]] |
| **GPU シェーダー詳細** | [[detail_radiosity]] / [[ref_radiosity]] |

---

### [4] Mesh SDF Culling（共有トレース基盤）

| 項目 | 内容 |
|-----|------|
| **概要** | Screen Probe・Radiance Cache・Radiosity が共用するトレース前カリング。Mesh SDF と Global SDF の2系統 |
| **CPU 関数** | 各トレース関数内（`RenderLumenScreenProbeGather()` 等）から呼ばれる |
| **シェーダー** | `LumenMeshSDFCulling.usf` / `LumenScene.usf` |
| **出力** | `VisibleMeshSDFsBuffer`（コンパクト化済み SDF リスト） |
| **CPU 詳細** | [[c_lumen_tracing]] |
| **GPU シェーダー詳細** | [[detail_sdf_tracing]] / [[ref_sdf_tracing]] |

---

### [5] Radiance Cache Update

| 項目 | 内容 |
|-----|------|
| **概要** | 3D 空間に疎配置したプローブキャッシュを更新。Screen Probe が届かない遠距離・カメラ外の間接光を補う |
| **CPU 関数** | `LumenRadianceCache::Update()` / `LumenRadianceCache::Render()` |
| **シェーダー** | `LumenRadianceCacheUpdate.usf`（Probe 管理 CS 群）, `LumenRadianceCache.usf#InterpolateRadianceCacheCS` |
| **出力** | Radiance Cache Atlas（球面 Irradiance テクスチャ） |
| **CPU 詳細** | [[d_lumen_radiance_cache]] |
| **GPU シェーダー詳細** | [[detail_radiance_cache]] / [[ref_radiance_cache]] |

---

### [6] Screen Probe Gather（AsyncCompute）

| 項目 | 内容 |
|-----|------|
| **概要** | スクリーンに疎に配置した Screen Probe からレイをトレースし、Diffuse GI テクスチャを生成 |
| **CPU 関数** | `RenderLumenScreenProbeGather()` (`LumenScreenProbeGather.cpp`) |
| **シェーダー** | `LumenScreenProbeTracing.usf` / `LumenScreenProbeGather.usf` / `LumenScreenProbeGatherTemporal.usf` / `LumenScreenProbeFiltering.usf` |
| **キュー** | AsyncCompute（GBuffer 描画と並列） |
| **出力** | `FSSDSignalTextures::Textures[0-2]`（Diffuse GI バッファ群） |
| **CPU 詳細** | [[e_lumen_diffuse_gi]] |
| **GPU シェーダー詳細** | [[detail_screen_probe_gather]] / [[ref_screen_probe_gather]] |

---

### [7] ReSTIR GI + Bent Normal（オプション）

| 項目 | 内容 |
|-----|------|
| **概要** | ReSTIR ベースの GI サンプリング（高品質オプション）と Bent Normal による短距離 AO 計算 |
| **CPU 関数** | `RenderLumenReSTIRGather()` / `RenderLumenScreenSpaceBentNormal()` |
| **シェーダー** | `LumenReSTIRGather.usf` / `LumenScreenSpaceBentNormal.usf` |
| **有効条件** | `r.Lumen.ReSTIR=1` / `r.Lumen.ScreenBentNormal=1` |
| **出力** | Diffuse GI バッファ（ReSTIR 時）/ Bent Normal テクスチャ |
| **CPU 詳細** | [[e_lumen_diffuse_gi]] |
| **GPU シェーダー詳細** | [[ref_restir_bent_normal]] |

---

### [8] Reflection Tracing + Resolve + Denoise（AsyncCompute）

| 項目 | 内容 |
|-----|------|
| **概要** | Roughness 別にレイをトレース → Resolve（再構成）→ テンポラル/スペーシャルデノイズで反射テクスチャを生成 |
| **CPU 関数** | `RenderLumenReflections()` |
| **シェーダー** | `LumenReflections.usf` / `LumenReflectionTracing.usf` / `LumenReflectionResolve.usf` / `LumenReflectionDenoiserTemporal.usf` / `LumenReflectionDenoiserSpatial.usf` |
| **キュー** | AsyncCompute（`ReflectionsMethod == Lumen` かつ AsyncCompute 有効時） |
| **出力** | `FSSDSignalTextures::Textures[3]`（反射バッファ） |
| **CPU 詳細** | [[f_lumen_reflections]] |
| **GPU シェーダー詳細** | [[detail_reflections]] / [[ref_reflection_tracing]] / [[ref_reflection_denoiser]] |

---

### [9] Final Composite

| 項目 | 内容 |
|-----|------|
| **概要** | Diffuse GI + 反射 + AO を GBuffer と合算して SceneColor に加算合成 |
| **CPU 関数** | `RenderDiffuseIndirectAndAmbientOcclusion()` (`IndirectLightRendering.cpp:977`) |
| **シェーダー** | `DiffuseIndirectComposite.usf#MainPS()`（`FDiffuseIndirectCompositePS`） |
| **出力** | `SceneColor`（GI + 反射込みの最終ライティング） |
| **CPU 詳細** | [[g_lumen_final_composite]] |
| **GPU シェーダー詳細** | [[detail_diffuse_indirect_composite]] / [[ref_diffuse_indirect_composite]] |

---

## AsyncCompute との関係

Lumen の GI・反射計算は GPU の Graphics キューと Compute キューで並列動作できる。

```
Graphics キュー:
  [5a] Card Capture
  [5b] Surface Cache Lighting（Direct + Radiosity）
  [6]  Base Pass（GBuffer 書き込み）← 重い GPU 処理

AsyncCompute キュー（r.Lumen.AsyncCompute=1 時）:
  ─── GBuffer と並列 ────────────────────────────
  [6]  Screen Probe Gather（Diffuse GI）
  [7]  Reflections（ReflectionsMethod == Lumen 時）
  ───────────────────────────────────────────────
  ※ GBuffer 完了後に FD3D12Queue::WaitForOtherQueue() でフェンス同期
  ※ Radiance Cache 更新も r.Lumen.RadianceCache.AsyncCompute=1 で非同期化可
```

---

## シェーダー別 CPU 対応一覧

| シェーダーファイル | 主要エントリポイント | CPU 関数 | グループ |
|----------------|----------------|---------|---------|
| `LumenCardVertexShader.usf` | `Main()` | `RenderCardCaptureViews()` | [[a_SurfaceCache]] |
| `LumenCardPixelShader.usf` | `Main()` | `RenderCardCaptureViews()` | [[a_SurfaceCache]] |
| `LumenCardComputeShader.usf` | `Main()` | `RenderCardCaptureViews()` | [[a_SurfaceCache]] |
| `SurfaceCache/LumenSurfaceCache.usf` | 複数 CS | `UpdateLumenScene()` | [[a_SurfaceCache]] |
| `LumenSceneDirectLightingCulling.usf` | `LumenSceneDirectLightingCullingCS()` | `RenderDirectLightingForLumenScene()` | [[b_SceneLighting]] |
| `LumenSceneDirectLightingShadowMask.usf` | `LumenSceneDirectLightingShadowMaskCS()` | `RenderDirectLightingForLumenScene()` | [[b_SceneLighting]] |
| `LumenSceneDirectLighting.usf` | `LumenCardBatchDirectLightingCS()` | `RenderDirectLightingForLumenScene()` | [[b_SceneLighting]] |
| `LumenSceneLighting.usf` | `CombineLumenSceneLightingCS()` | `RenderLumenSceneLighting()` | [[b_SceneLighting]] |
| `Radiosity/LumenRadiosity.usf` | 複数 CS（Mark / Trace / Filter / Integrate） | `RenderRadiosityForLumenScene()` | [[b_SceneLighting]] |
| `LumenMeshSDFCulling.usf` | `CullMeshSDFObjectsForViewCS()` / `CompactCulledObjectsCS()` | 各トレース関数 | [[c_Tracing]] |
| `LumenScene.usf` | `LumenCardUpdateCS()` など | `UpdateLumenScene()` | [[c_Tracing]] |
| `LumenRadianceCacheUpdate.usf` | `AllocateUsedProbesCS()` / `TraceFromProbesCS()` など | `LumenRadianceCache::Update()` | [[d_RadianceCache]] |
| `LumenRadianceCache.usf` | `InterpolateRadianceCacheCS()` | `LumenRadianceCache::Render()` | [[d_RadianceCache]] |
| `LumenScreenProbeTracing.usf` | `LumenScreenProbeTraceCS()` | `RenderLumenScreenProbeGather()` | [[e_DiffuseGI]] |
| `LumenScreenProbeGather.usf` | `LumenScreenProbeGatherCS()` | `RenderLumenScreenProbeGather()` | [[e_DiffuseGI]] |
| `LumenScreenProbeGatherTemporal.usf` | `TemporallyAccumulateScreenProbesCS()` | `RenderLumenScreenProbeGather()` | [[e_DiffuseGI]] |
| `LumenScreenProbeFiltering.usf` | `FilterScreenProbesCS()` | `RenderLumenScreenProbeGather()` | [[e_DiffuseGI]] |
| `LumenReSTIRGather.usf` | `ReSTIRGatherCS()` など | `RenderLumenReSTIRGather()` | [[e_DiffuseGI]] |
| `LumenScreenSpaceBentNormal.usf` | `LumenScreenSpaceBentNormalCS()` | `RenderLumenScreenSpaceBentNormal()` | [[e_DiffuseGI]] |
| `LumenReflections.usf` | `ReflectionTileClassificationBuildListsCS()` / `ReflectionGenerateRaysCS()` | `RenderLumenReflections()` | [[f_Reflections]] |
| `LumenReflectionTracing.usf` | `ReflectionTraceScreenTexturesCS()` / `ReflectionTraceMeshSDFsCS()` など | `RenderLumenReflections()` | [[f_Reflections]] |
| `LumenReflectionResolve.usf` | `LumenReflectionResolveCS()` | `RenderLumenReflections()` | [[f_Reflections]] |
| `LumenReflectionDenoiserTemporal.usf` | `LumenReflectionDenoiserTemporalCS()` | `RenderLumenReflections()` | [[f_Reflections]] |
| `LumenReflectionDenoiserSpatial.usf` | `LumenReflectionDenoiserSpatialCS()` | `RenderLumenReflections()` | [[f_Reflections]] |
| `DiffuseIndirectComposite.usf` | `MainPS()` | `RenderDiffuseIndirectAndAmbientOcclusion()` | [[g_FinalComposite]] |
