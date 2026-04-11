# PostProcess ドキュメント作成チェックリスト

## Details（グループ別概要 7本）

- [x] `a_pp_orchestrator.md`   … AddPostProcessingPasses() / FPostProcessingInputs
- [x] `b_pp_taa_tsr.md`        … TemporalAA / TemporalSuperResolution
- [x] `c_pp_bloom.md`          … Bloom（Setup + FFT + WeightedSampleSum + Downsample）
- [x] `d_pp_exposure.md`       … 露出（Histogram + EyeAdaptation + LocalExposure）
- [x] `e_pp_dof_motionblur.md` … 被写界深度（DiaphragmDOF + BokehDOF）+ MotionBlur
- [x] `f_pp_tonemap.md`        … Tonemap + CombineLUTs + ACESUtils
- [x] `g_pp_misc.md`           … Material / Subsurface / LensFlares / Upscale / Neural / AA

## Reference（ファイル別 12本）

### 主要パイプライン（8本）
- [x] `ref_pp_orchestrator.md` … PostProcessing.h/.cpp
- [x] `ref_pp_taa.md`          … TemporalAA.h/.cpp
- [x] `ref_pp_tsr.md`          … TemporalSuperResolution.cpp（ヘッダなし）
- [x] `ref_pp_bloom.md`        … BloomSetup + FFTBloom + WeightedSampleSum + Downsample
- [x] `ref_pp_exposure.md`     … Histogram + EyeAdaptation + LocalExposure
- [x] `ref_pp_dof.md`          … DiaphragmDOF + BokehDOF + PostProcessDOF
- [x] `ref_pp_motionblur.md`   … PostProcessMotionBlur.h/.cpp
- [x] `ref_pp_tonemap.md`      … Tonemap + CombineLUTs + ACESUtils

### まとめ系（4本）
- [x] `ref_pp_material_misc.md`  … Material / Subsurface / LensFlares / Upscale / Neural / AA / LensDistortion / MitchellNetravali / AlphaInvert / SceneFilterRendering
- [x] `ref_pp_visualize.md`      … 全Visualize系（GBufferHints / VisualizeBuffer / VisualizeShadingModels 等）
- [x] `ref_pp_debug_editor.md`   … Debug/Editor系（BufferInspector / CompositeEditor / SelectionOutline 等）
- [x] `ref_pp_platform.md`       … プラットフォーム系（Mobile / HMD / DeviceEncodingOnly / SMAA 等）

---

## Phase 3（コード実行フロー + callout 追加）

### Overview（1本）
- [x] `05_postprocess_overview.md` … `## コード実行フロー` 追加

### Details（7本） — `## コード実行フロー` + `## 関連リファレンス` 追加
- [x] `a_pp_orchestrator.md`
- [x] `b_pp_taa_tsr.md`
- [x] `c_pp_bloom.md`
- [x] `d_pp_exposure.md`
- [x] `e_pp_dof_motionblur.md`
- [x] `f_pp_tonemap.md`
- [x] `g_pp_misc.md`

### Reference（12本） — `> [!note]-` callout × 3 追加
- [x] `ref_pp_orchestrator.md`
- [x] `ref_pp_taa.md`
- [x] `ref_pp_tsr.md`
- [x] `ref_pp_bloom.md`
- [x] `ref_pp_exposure.md`
- [x] `ref_pp_dof.md`
- [x] `ref_pp_motionblur.md`
- [x] `ref_pp_tonemap.md`
- [x] `ref_pp_material_misc.md`
- [x] `ref_pp_visualize.md`
- [x] `ref_pp_debug_editor.md`
- [x] `ref_pp_platform.md`
