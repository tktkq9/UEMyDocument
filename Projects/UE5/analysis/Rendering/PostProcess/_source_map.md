# PostProcess ソースマップ

- 対象: ポストプロセスパイプライン全体（TAA/TSR, Bloom, Exposure, DOF, Tonemap, ...）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[05_postprocess_overview]]

`AddPostProcessingPasses()` が全パスを RDG に順番登録するオーケストレーター。
MotionBlur → TAA/TSR → Bloom → Exposure → DOF → Tonemap → Upscale の順で SceneColor(HDR) から最終出力へ。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/PostProcess/` |
| 公開 API | `Engine/Source/Runtime/Renderer/Public/TemporalUpscaler.h`, `PostProcess/PostProcessing.h` |
| シェーダー | `Engine/Shaders/Private/PostProcess*.usf`, `TSR/*.usf` |
| 呼び出し元 | `Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp` |

---

## ファイル → クラス対応

### オーケストレーター

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `PostProcess/PostProcessing.h/.cpp` | `AddPostProcessingPasses()`:347, `AddDebugViewPostProcessingPasses()`:2073, `FPostProcessingInputs` | 全パス登録・デバッグビュー | [[Reference/ref_pp_orchestrator]], [[Details/a_pp_orchestrator]] |

### TAA / TSR

| ファイル | 主要関数 / クラス | 役割 | 参照 |
|---------|----------------|------|------|
| `PostProcess/TemporalAA.h/.cpp` | `AddTemporalAAPass()`:571, `ITemporalUpscaler` | TAA 実行・TSR/DLSS 公開 IF | [[Reference/ref_pp_taa]] |
| `PostProcess/TemporalSuperResolution.cpp` | TSR 実装 | アップスケーリング付きテンポラル | [[Reference/ref_pp_tsr]], [[Details/b_pp_taa_tsr]] |
| `Public/TemporalUpscaler.h` | `ITemporalUpscaler::AddPasses()` | プラグイン差し替え用 API | 同 |

### Bloom / Lens

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `PostProcess/PostProcessBloomSetup.h/.cpp` | `AddBloomSetupPass()`:120 | ガウシアン Bloom | [[Reference/ref_pp_bloom]], [[Details/c_pp_bloom]] |
| `PostProcess/PostProcessFFTBloom.h/.cpp` | — | FFT 高品質 Bloom | 同 |
| `PostProcess/PostProcessLensFlares.h/.cpp` | — | レンズフレア | — |

### Exposure

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `PostProcess/PostProcessHistogram.h/.cpp` | `AddHistogramPass()`:451 | 輝度ヒストグラム | [[Reference/ref_pp_exposure]], [[Details/d_pp_exposure]] |
| `PostProcess/PostProcessEyeAdaptation.h/.cpp` | `AddHistogramEyeAdaptationPass()`:1023 | 自動露出 | 同 |
| `PostProcess/PostProcessLocalExposure.h/.cpp` | — | ローカル露出 | 同 |

### DOF / MotionBlur

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `PostProcess/DiaphragmDOF.h/.cpp` | `DiaphragmDOF::AddPasses()`:1486 | 高品質 DOF | [[Reference/ref_pp_dof]], [[Details/e_pp_dof_motionblur]] |
| `PostProcess/PostProcessBokehDOF.cpp` | — | ボケシェイプ DOF | 同 |
| `PostProcess/PostProcessMotionBlur.h/.cpp` | `AddMotionBlurPass()`:1333 | モーションブラー | [[Reference/ref_pp_motionblur]] |

### Tonemap

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `PostProcess/PostProcessCombineLUTs.h/.cpp` | `AddCombineLUTPass()`:494 | Color LUT 合成 | [[Reference/ref_pp_tonemap]], [[Details/f_pp_tonemap]] |
| `PostProcess/PostProcessTonemap.h/.cpp` | `AddTonemapPass()`:569 | トーンマッパー（ACES 等） | 同 |

### その他

| ファイル | 役割 | 参照 |
|---------|------|------|
| `PostProcess/PostProcessAA.cpp` | FXAA 等簡易 AA | [[Reference/ref_pp_material_misc]] |
| `PostProcess/PostProcessUpscale.h/.cpp` | 単純アップスケール | [[Details/g_pp_misc]] |
| `PostProcess/PostProcessSubsurface.h/.cpp` | サブサーフェス散乱後処理 | 同 |
| `PostProcess/PostProcessMaterial.h/.cpp` | ユーザーカスタム PP マテリアル | [[Reference/ref_pp_material_misc]] |
| `PostProcess/NeuralPostProcess.h/.cpp` | ニューラル後処理（実験的） | 同 |

### Visualize / Debug

| ファイル | 役割 | 参照 |
|---------|------|------|
| `PostProcess/VisualizeBuffer.cpp` | デバッグビュー | [[Reference/ref_pp_visualize]], [[Reference/ref_pp_debug_editor]] |
| `PostProcess/PostProcessSelectionOutline.cpp` | 選択アウトライン（Editor） | 同 |
| `PostProcess/PostProcessGBufferHints.cpp` | GBuffer ヒント | 同 |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()
  │
  └─ AddPostProcessingPasses(View, Inputs, ...)       PostProcessing.cpp:347
       │
       ├─[A] AddMotionBlurPass                        PostProcessMotionBlur.cpp:1333
       │
       ├─[B] AddTemporalAAPass                        TemporalAA.cpp:571
       │     or ITemporalUpscaler::AddPasses          TemporalSuperResolution.cpp
       │
       ├─[C] AddBloomSetupPass + ガウシアン/FFT       PostProcessBloomSetup.cpp:120
       │
       ├─[D] AddHistogramPass                         PostProcessHistogram.cpp:451
       │     AddHistogramEyeAdaptationPass            PostProcessEyeAdaptation.cpp:1023
       │
       ├─[E] DiaphragmDOF::AddPasses                  DiaphragmDOF.cpp:1486
       │
       ├─[F] AddCombineLUTPass                        PostProcessCombineLUTs.cpp:494
       │     AddTonemapPass                           PostProcessTonemap.cpp:569
       │
       ├─[G] AddPostProcessUpscalePass                PostProcessUpscale.cpp
       │
       └─[H] AddDebugViewPostProcessingPasses         PostProcessing.cpp:2073
```

---

## 主要データ構造

```cpp
// FPostProcessingInputs（PostProcessing.h）
struct FPostProcessingInputs
{
    FRDGTextureRef SceneColor;
    FRDGTextureRef SceneDepth;
    FRDGTextureRef SceneVelocity;
    FRDGTextureRef SeparateTranslucency;
    TRDGUniformBufferRef<FSceneTextureUniformParameters> SceneTextures;
};

// ITemporalUpscaler（TemporalUpscaler.h）: TSR / DLSS / XeSS の差し替え IF
class ITemporalUpscaler
{
    virtual FOutputs AddPasses(FRDGBuilder&, const FViewInfo&, const FPassInputs&) const = 0;
};
```

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.AntiAliasingMethod` | `PostProcessing.cpp`（0=None, 1=FXAA, 2=TAA, 4=TSR）|
| `r.TemporalAA.Quality` | `TemporalAA.cpp` |
| `r.TSR.History.ScreenPercentage` | `TemporalSuperResolution.cpp` |
| `r.BloomQuality` | `PostProcessBloomSetup.cpp` |
| `r.DepthOfFieldQuality` | `DiaphragmDOF.cpp` |
| `r.Tonemapper.Quality` | `PostProcessTonemap.cpp` |
| `r.EyeAdaptation.MethodOverride` | `PostProcessEyeAdaptation.cpp` |
| `r.LocalExposure.HighlightContrastScale` | `PostProcessLocalExposure.cpp` |
| `r.MotionBlurQuality` | `PostProcessMotionBlur.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_pp_orchestrator]] | AddPostProcessingPasses |
| Reference | [[Reference/ref_pp_taa]] | TemporalAA |
| Reference | [[Reference/ref_pp_tsr]] | TemporalSuperResolution |
| Reference | [[Reference/ref_pp_bloom]] | Bloom |
| Reference | [[Reference/ref_pp_exposure]] | Histogram / EyeAdaptation |
| Reference | [[Reference/ref_pp_dof]] | DOF |
| Reference | [[Reference/ref_pp_motionblur]] | MotionBlur |
| Reference | [[Reference/ref_pp_tonemap]] | Tonemap / LUT |
| Reference | [[Reference/ref_pp_material_misc]] | PostProcessMaterial / FXAA / Neural |
| Reference | [[Reference/ref_pp_visualize]] | VisualizeBuffer |
| Reference | [[Reference/ref_pp_debug_editor]] | 選択アウトライン等 |
| Reference | [[Reference/ref_pp_platform]] | プラットフォーム固有 |
| Details | [[Details/a_pp_orchestrator]] | オーケストレーション |
| Details | [[Details/b_pp_taa_tsr]] | TAA/TSR 実装 |
| Details | [[Details/c_pp_bloom]] | Bloom 実装 |
| Details | [[Details/d_pp_exposure]] | Exposure 実装 |
| Details | [[Details/e_pp_dof_motionblur]] | DOF / MotionBlur |
| Details | [[Details/f_pp_tonemap]] | Tonemap 実装 |
| Details | [[Details/g_pp_misc]] | その他小規模パス |

---

## ue5-dive 起点

- 「PP パスの順序」 → `PostProcessing.cpp:AddPostProcessingPasses:347`
- 「TAA と TSR の切替」 → `r.AntiAliasingMethod` + `ITemporalUpscaler` 実装
- 「DLSS / XeSS 差し替え」 → `ITemporalUpscaler::AddPasses` を実装するプラグイン
- 「Tonemap の ACES 実装」 → `PostProcessTonemap.cpp` + `Tonemapper.usf`
- 「EyeAdaptation の履歴」 → `PostProcessEyeAdaptation.cpp:AddHistogramEyeAdaptationPass:1023`
- 「PP マテリアルの挿入位置」 → `PostProcessMaterial.cpp` + `BL_*` (BlendableLocation) enum
