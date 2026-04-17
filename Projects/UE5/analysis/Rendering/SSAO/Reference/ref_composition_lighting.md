# ref: FCompositionLighting / AddAmbientOcclusionPasses / PostProcessingInputs

- 対象ファイル: `CompositionLighting/CompositionLighting.h/.cpp` / `PostProcessAmbientOcclusion.cpp`
- 概要: [[22_ssao_overview]]

---

## CompositionLighting の概要

`FCompositionLighting` は GBuffer 後の **AO / SSS / GI を統合するフレームワーク**。  
`ProcessAfterOcclusion()` がメインエントリーポイントで、  
AO 方式の選択・テクスチャ生成・SceneTextures への登録を行う。

---

## ProcessAfterOcclusion() シグネチャ

```cpp
// CompositionLighting.h
namespace CompositionLighting
{
    void ProcessAfterOcclusion(
        FRDGBuilder& GraphBuilder,
        const FViewInfo& View,
        const FSceneTextures& SceneTextures,
        FScreenPassTexture& ScreenPassAO,
        bool bCompositeRegularLighting);
}
```

---

## AddAmbientOcclusionPasses()（PostProcessAmbientOcclusion.cpp）

```cpp
FRDGTextureRef AddAmbientOcclusionPasses(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FGTAOContext& GTAOContext,
    const FSceneTextures& SceneTextures,
    FRDGTextureRef* OutBentNormal = nullptr);

// ─── 戻り値 ─────────────────────────────────────────────────
// return: AO テクスチャ（R8 or R8G8）
// OutBentNormal: BentNormal テクスチャ（あれば）
//   → IBL の Specular Occlusion に使用可能

// ─── GTAOContext が決める処理 ──────────────────────────────
// EGTAOType::ENonAsync:
//   全パスを同期的に GFX パイプで実行
// EGTAOType::EAsyncHorizonSearch:
//   HorizonSearch を Async Compute で起動（FRDGAsyncComputeBudget::EAll）
// EGTAOType::EAsyncCombinedSpatial:
//   HorizonSearch + Integrate + SpatialFilter を Async Compute
```

---

## FPostProcessingInputs（PostProcessing.h）

```cpp
// AddPostProcessingPasses() に渡す入力構造体
struct FPostProcessingInputs
{
    FRDGTextureRef SceneColor = nullptr;         // メイン SceneColor
    FRDGTextureRef SceneDepth = nullptr;         // SceneDepth
    FRDGTextureRef SeparateTranslucency = nullptr;
    FRDGTextureRef ScreenSpaceAO = nullptr;      // CompositionLighting が生成した AO
    FRDGTextureRef SceneVelocity = nullptr;
    // ... 他フィールド ...
};
```

---

## AO と DeferredLighting の統合

```
// DeferredLightPixelShaders.usf（DeferredLightingComposite）

// GBuffer から AO を取得
float AO = 1.0f;
if (bHasScreenSpaceAO)
{
    AO = ScreenSpaceAO.SampleLevel(ScreenUV, 0).r;
    // R8 フォーマット: 0.0 = 完全遮蔽, 1.0 = 遮蔽なし
}

// 間接拡散光に AO を適用
IndirectIrradiance *= AO;

// 間接スペキュラ（反射）にも AO を適用
SpecularOcclusion = pow(AO, SpecularOcclusionExponent);
// r.DFAO.SpecularOcclusionMode:
//   0 = スペキュラに AO 非適用
//   1 = AO をスペキュラにも適用

// GTAO BentNormal（利用可能な場合）
// → IBL サンプリング方向を BentNormal に曲げて遮蔽を考慮
```

---

## DFAO（Distance Field AO）との共存

```cpp
// DistanceFieldAmbientOcclusion.h
struct FDistanceFieldAOParameters
{
    float OcclusionMaxDistance; // 遮蔽の最大距離（デフォルト 600cm）
    float Contrast;             // コントラスト強調
};

// 実行条件:
// r.GenerateMeshDistanceFields = 1
// r.AOWithSupportForSkyOcclusion = 1
// Lumen 無効

// SSAO/GTAO との違い:
// DFAO: 長距離遮蔽（中〜遠距離）を 3D 距離フィールドから計算
// GTAO: 短〜中距離の遮蔽をスクリーン空間で計算
// → 両方有効な場合は DFAO が GTAO を置き換える（より高品質）
// → Lumen 有効なら Lumen Short Range AO が両方を置き換える
```

---

## AO 適用の全体フロー（フレーム内位置）

```
InitViews()
  AllocateOcclusionTests()        // GPU Occlusion Query 準備

RenderPrePass()
  BuildHZB()                      // HZB 構築（AO の HorizonSearch に使用）

BasePass() → GBuffer 完成

CompositionLighting::ProcessAfterOcclusion()
  AddAmbientOcclusionPasses()     // GTAO/SSAO/DFAO のいずれかを実行
  → SceneTextures.ScreenSpaceAO = AOTexture 登録

RenderLights()
  DeferredLightingComposite
  → AO テクスチャを間接光に乗算
```

---

## 関連リファレンス

- [[ref_ssao]] — `FGTAOContext` / `EGTAOType` / シェーダークラス
- [[b_composition_lighting]] — CompositionLighting フロー詳細
