# GPU PostProcess ソースマップ

- 対象: PostProcess GPU シェーダー（TSR/TAA/Bloom/DOF/MotionBlur/EyeAdaptation/Tonemap）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_postprocess_gpu_overview]]

`AddPostProcessingPasses()` がメインエントリ。RDG 上に TSR/TAA → EyeAdaptation → Bloom → DOF → MotionBlur → Tonemap の順にパスを構築。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/PostProcess/` |
| シェーダー | `Engine/Shaders/Private/PostProcess*.usf` / `TemporalAA/*.usf` / `TSR/*.usf` |

---

## ファイル → シェーダー対応

### アップスケール・AA

| CPU ファイル | 主要関数 | シェーダー | 役割 | 参照 |
|-----------|--------|----------|------|------|
| `TemporalSuperResolution.cpp` | `AddTemporalSuperResolutionPasses()` | `TSR/*.usf` | History Reprojection / Update / Upscale / Nyquist | [[detail_tsr]] |
| `TemporalAA.h/.cpp` | `AddTemporalAAPasses()` | `TAA/*.usf` | ジッタリング + History ブレンド | [[detail_taa]] |

### Bloom / EyeAdaptation / Tonemap

| CPU ファイル | 主要関数 | シェーダー | 役割 | 参照 |
|-----------|--------|----------|------|------|
| `PostProcessBloomSetup.cpp` | `AddBloomSetupPass()` | `PostProcessBloomSetup.usf` | 閾値 + Downsample | [[detail_bloom]] |
| `PostProcessFFTBloom.cpp` | — | `PostProcessFFTBloom.usf` | FFT Bloom（高品質） | 同 |
| `PostProcessLensFlares.cpp` | `AddLensFlaresPass()` | `PostProcessLensFlares.usf` | Lens Flare | 同 |
| `PostProcessHistogram.cpp` | `AddHistogramPass()` | `PostProcessHistogram.usf` | 輝度ヒストグラム | [[detail_tonemap]] |
| `PostProcessEyeAdaptation.cpp` | `AddEyeAdaptationPass()` | `PostProcessEyeAdaptation.usf` | 露出値更新 | 同 |
| `PostProcessLocalExposure.cpp` | — | `PostProcessLocalExposure.usf` | 局所露出 | 同 |
| `PostProcessCombineLUTs.cpp` | — | `PostProcessCombineLUTs.usf` | Color Grading LUT 合成 | 同 |
| `PostProcessTonemap.cpp` | `AddTonemapPass()` | `PostProcessTonemap.usf` | ACES + Gamma + LUT | 同 |
| `ACESUtils.h/.cpp` | — | `ACES.ush` | ACES 変換ユーティリティ | 同 |

### DOF / MotionBlur

| CPU ファイル | 主要関数 | シェーダー | 役割 | 参照 |
|-----------|--------|----------|------|------|
| `DiaphragmDOF.h/.cpp` | `AddDiaphragmDOFPasses()` | `DiaphragmDOF/*.usf` | CoC 計算・Near/Far 分離・Gather・Recombine | [[detail_dof]] |
| `PostProcessMotionBlur.h/.cpp` | `AddMotionBlurPass()` | `PostProcessMotionBlur.usf` | Velocity Flatten / Tile / Blur（Scatter-as-Gather） | [[detail_motion_blur]] |

### 統合

| CPU ファイル | 主要関数 | 役割 |
|-----------|--------|------|
| `PostProcessing.h/.cpp` | `AddPostProcessingPasses()` | 全パス RDG 上で統合 |

---

## GPU データフロー

```
AddPostProcessingPasses()
  ├─[A] TSR / TAA                    TemporalSuperResolution.cpp / TemporalAA.cpp
  │     → アップスケール + AA
  ├─[B] Histogram + EyeAdaptation    PostProcessHistogram.cpp / PostProcessEyeAdaptation.cpp
  │     → 露出値
  ├─[C] Bloom Setup + Gaussian       PostProcessBloomSetup.cpp
  │     → 閾値 → Downsample → Upsample
  ├─[D] DOF（Diaphragm）             DiaphragmDOF.cpp
  │     → CoC → Near/Far 分離 → Gather → Recombine
  ├─[E] Motion Blur                  PostProcessMotionBlur.cpp
  │     → Velocity Flatten → Tile → Blur
  └─[F] Tonemap                      PostProcessTonemap.cpp / ACESUtils.cpp
        → Bloom 合成 → ACES → LUT → Gamma → 出力
```

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.TSR.*` | `TemporalSuperResolution.cpp` |
| `r.TemporalAA.*` | `TemporalAA.cpp` |
| `r.Bloom.Quality` | `PostProcessBloomSetup.cpp` |
| `r.EyeAdaptationQuality` | `PostProcessEyeAdaptation.cpp` |
| `r.Tonemapper.*` | `PostProcessTonemap.cpp` |
| `r.DOF.*` | `DiaphragmDOF.cpp` |
| `r.MotionBlur.*` | `PostProcessMotionBlur.cpp` |

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_tsr]] / [[detail_taa]] / [[detail_bloom]] / [[detail_tonemap]] / [[detail_dof]] / [[detail_motion_blur]] |

---

## ue5-dive 起点

- 「PostProcess 全体エントリ」 → `PostProcessing.cpp:AddPostProcessingPasses`
- 「TSR 本体」 → `TemporalSuperResolution.cpp:AddTemporalSuperResolutionPasses`
- 「Bloom パイプライン」 → `PostProcessBloomSetup.cpp`
- 「DOF」 → `DiaphragmDOF.cpp:AddDiaphragmDOFPasses`
- 「Tonemap + ACES」 → `PostProcessTonemap.cpp` + `ACESUtils.cpp`
