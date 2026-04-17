# b: CompositionLighting（AO / GI 合成フレームワーク）

- 対象ファイル: `CompositionLighting/CompositionLighting.h/.cpp`
- 概要: [[22_ssao_overview]]

---

## 概要

`CompositionLighting` は **GBuffer 後の後処理として AO・GI・SSS 等を合成するフレームワーク**。  
`ProcessAfterOcclusion()` で AO を計算し、`ProcessLpvIndirect()` で LPV（旧方式）の  
間接光を合成する。UE5 では主に SSAO / GTAO の統合に使われる。

---

## CompositionLighting の役割

```
FDeferredShadingSceneRenderer::Render() における位置:

[BasePass] → [GBuffer 完成] → [Shadow Depth] → [DeferredLighting]
                                    ↓
                        CompositionLighting::ProcessAfterOcclusion()
                          → SSAO / GTAO / Distance Field AO 計算
                          → AO テクスチャ生成
                                    ↓
                        [DeferredLighting: AO テクスチャを参照]
```

---

## ProcessAfterOcclusion() フロー

```
FCompositionLighting::ProcessAfterOcclusion(GraphBuilder, Views, ...)
  │
  ├─ for each View:
  │
  ├─ [A] AO 方式の判定
  │   if (Lumen が AO を提供する)
  │     → LumenShortRangeAO からテクスチャを取得（スキップ）
  │   else if (DistanceField AO が有効)
  │     → AddDistanceFieldAOPasses() でDFAO を計算
  │   else
  │     → AddAmbientOcclusionPasses() で SSAO/GTAO を計算
  │
  ├─ [B] AO テクスチャの登録
  │   SceneTextures.ScreenSpaceAO = AOTexture
  │   → DeferredLightingComposite でサンプル
  │
  └─ [C] SubsurfaceScattering（Separable SSS）
      bSubsurfaceLightingEnabled → AddSubsurfacePass()
      → GBufferB の SSS Profile を参照して拡散

【DeferredLighting での AO 利用】
  DeferredLightPixelShaders.usf:
    float AO = ScreenSpaceAO.SampleLevel(ScreenUV, 0).r
    → IndirectLighting *= AO  // 間接光に AO を乗算
    → SpecularColor *= pow(AO, SpecularOcclusionExponent)  // スペキュラにも
```

---

## GTAO / SSAO の切り替えロジック

```cpp
// PostProcessAmbientOcclusion.cpp の GetGTAOType():

if (!IsAmbientOcclusionCompute(FeatureLevel))
    return EGTAOType::EOff;  // SM5 以下は SSAO にフォールバック

if (r.AmbientOcclusion.Method == 0)
    return EGTAOType::EOff;  // SSAO モード（Method=0）

if (r.GTAO.Async == 1 && r.GTAO.HorizonSearchIntegrate == 1)
    return EGTAOType::EAsyncCombinedSpatial;

if (r.GTAO.Async == 1)
    return EGTAOType::EAsyncHorizonSearch;

return EGTAOType::ENonAsync;
```

---

## AddPostProcessingPasses() 内での AO 統合

```
PostProcessing.cpp / AddPostProcessingPasses():
  │
  ├─ [Pre-PostProcess]
  │   CompositionLighting::ProcessAfterOcclusion()
  │     → ScreenSpaceAO テクスチャ確定
  │
  ├─ [DeferredLighting]
  │   RenderLights() → DeferredLightingComposite
  │     → ScreenSpaceAO を間接光に乗算
  │
  └─ [PostProcess]
      AO Visualize パス（r.VisualizeAO=1 時）
      → デバッグ表示
```

---

## Distance Field AO との関係

```
DFAO（Distance Field Ambient Occlusion）:
  r.GenerateMeshDistanceFields=1 かつ r.AOWithSupportForSkyOcclusion=1 が必要

FDistanceFieldAOParameters:
  OcclusionMaxDistance: 距離フィールドの最大遮蔽距離
  Contrast:             AO のコントラスト強調

処理:
  CullObjectsForView() → 距離フィールドオブジェクトのカリング
  AddDistanceFieldAOPasses()
    → CS で3D距離フィールドテクスチャから遮蔽を計算
    → SSAO より長距離の AO を提供（中〜長距離遮蔽）
    → Lumen 有効時は不要（Lumen SSGI が同等以上を提供）
```

---

## 関連リファレンス

- [[ref_composition_lighting]] — `FCompositionLighting` / パス定義詳細
- [[a_ssao]] — GTAO / SSAO の詳細アルゴリズム
