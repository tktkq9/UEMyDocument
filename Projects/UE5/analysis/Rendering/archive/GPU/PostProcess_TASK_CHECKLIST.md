# Post-processing GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_postprocess_gpu_overview.md` … Post-processing GPU 全処理実行順 + CPU対応表

---

## Post-processing GPU シェーダー（グループ別）

### a: TSR（2本）
- [x] `a_TSR/detail_tsr.md`  … Temporal Super Resolution（History Reprojection + Upscale + Nyquist Filter）
- [x] `a_TSR/ref_tsr.md`     … TSRHistory / TSRReprojectHistory / TSRUpdateHistory / TSRVisualizeHistory エントリポイント

### b: TAA（2本）
- [x] `b_TAA/detail_taa.md`  … Temporal Anti-Aliasing（ジッタリング + History ブレンド + ゴースト抑制）
- [x] `b_TAA/ref_taa.md`     … TemporalAA MainCS / TemporalUpsample エントリポイント

### c: Bloom（2本）
- [x] `c_Bloom/detail_bloom.md`  … Bloom Downsample / Upsample / Lens Flare / Dirt Mask 合成
- [x] `c_Bloom/ref_bloom.md`     … BloomDownsample / BloomUpsample / LensFlareBlur エントリポイント

### d: Tonemap（2本）
- [x] `d_Tonemap/detail_tonemap.md`  … Exposure / Histogram / ACES Tone Mapping / Gamma Correction
- [x] `d_Tonemap/ref_tonemap.md`     … EyeAdaptationCS / TonemapPS / Histogram エントリポイント

### e: DOF（2本）
- [x] `e_DOF/detail_dof.md`  … Cinematic DOF（Near / Far Bokeh 分離 + Scatter-as-Gather）
- [x] `e_DOF/ref_dof.md`     … DiaphragmDOF SetupCS / GatherCS / RecombinePS エントリポイント

### f: MotionBlur（2本）
- [x] `f_MotionBlur/detail_motion_blur.md`  … Velocity Tile / Dilation + Scatter-as-Gather Motion Blur
- [x] `f_MotionBlur/ref_motion_blur.md`     … MotionBlur Velocity Flatten / MotionBlurPS エントリポイント

---

合計: 概要 1 + Post-processing シェーダー 12 = **13 ファイル** ✅
