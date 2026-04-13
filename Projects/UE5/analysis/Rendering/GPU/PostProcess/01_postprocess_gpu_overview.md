# Post-processing GPU 全体概要

- 取得日: 2026-04-13
- 対象: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\PostProcess\`
- 上位: [[01_gpu_overview]]
- CPU対応: [[05_postprocess_overview]]

---

## Post-processing とは

シーンカラー（HDR）に対して **アンチエイリアシング・被写界深度・モーションブラー・Bloom・トーンマップ** を順番に適用する後処理パイプライン。  
`AddPostProcessingPasses()` が全パスを RDG 上に構築するメインエントリポイント。

| パス | 役割 |
|------|------|
| TSR / TAA | テンポラルアップスケール・アンチエイリアシング |
| DOF | 被写界深度（Diaphragm DOF / Bokeh DOF） |
| Motion Blur | ベロシティタイル + Scatter-as-Gather |
| Bloom | Downsample → Upsample → LensFlare 合成 |
| Eye Adaptation | ヒストグラム → 露出値更新 |
| Tonemap | ACES / Gamma 補正 → LUT 適用 → 出力 |

---

## GPU 実行フロー（概略）

```
AddPostProcessingPasses()
  │
  ├─ [A] TSR or TAA                    // TemporalSuperResolution.cpp / TemporalAA.cpp
  │       アップスケール + ノイズ除去
  │
  ├─ [B] Histogram + EyeAdaptation     // PostProcessHistogram.cpp / PostProcessEyeAdaptation.cpp
  │       シーン輝度 → 露出値更新
  │
  ├─ [C] Bloom Setup + Gaussian        // PostProcessBloomSetup.cpp
  │       閾値 → Downsample チェーン → Upsample
  │
  ├─ [D] DOF（Diaphragm DOF）         // DiaphragmDOF.cpp
  │       CoC 計算 → Near/Far 分離 → Gather → Recombine
  │
  ├─ [E] Motion Blur                   // PostProcessMotionBlur.cpp
  │       Velocity Flatten → Tile → Blur
  │
  └─ [F] Tonemap                       // PostProcessTonemap.cpp / ACESUtils.cpp
          Bloom 合成 → ACES → LUT → Gamma → 出力
```

---

## CPU 対応表

| GPU シェーダーグループ | CPU 制御 | CVar 代表 |
|---------------------|---------|----------|
| TSR | `TemporalSuperResolution.cpp` | `r.TSR.*` |
| TAA | `TemporalAA.cpp` | `r.TemporalAA.*` |
| Bloom | `PostProcessBloomSetup.cpp` | `r.Bloom.Quality` |
| EyeAdaptation | `PostProcessEyeAdaptation.cpp` | `r.EyeAdaptationQuality` |
| Tonemap | `PostProcessTonemap.cpp` | `r.Tonemapper.*` |
| DOF | `DiaphragmDOF.cpp` | `r.DOF.*` |
| MotionBlur | `PostProcessMotionBlur.cpp` | `r.MotionBlur.*` |

---

## 主要ソースファイル一覧

| ファイル | 役割 |
|---------|------|
| `PostProcessing.h/.cpp` | `AddPostProcessingPasses()` — 全パス統合エントリ |
| `TemporalSuperResolution.cpp` | TSR 実装（History Reprojection / Update / Upscale） |
| `TemporalAA.h/.cpp` | TAA 実装（TAA / TAA Upsampling） |
| `PostProcessHistogram.h/.cpp` | ヒストグラム生成パス |
| `PostProcessEyeAdaptation.h/.cpp` | 自動露出（Eye Adaptation）パス |
| `PostProcessBloomSetup.h/.cpp` | Bloom セットアップ・閾値適用 |
| `PostProcessFFTBloom.h/.cpp` | FFT ベース Bloom（高品質モード） |
| `PostProcessLensFlares.h/.cpp` | Lens Flare パス |
| `PostProcessTonemap.h/.cpp` | トーンマップ・Gamma 補正 |
| `ACESUtils.h/.cpp` | ACES カラースペース変換ユーティリティ |
| `DiaphragmDOF.h/.cpp` | Cinematic DOF（物理ベース CoC） |
| `PostProcessMotionBlur.h/.cpp` | モーションブラー |
| `PostProcessLocalExposure.h/.cpp` | Local Exposure（局所露出） |
| `PostProcessCombineLUTs.h/.cpp` | Color Grading LUT 合成 |

---

## サブシステムドキュメント

| ドキュメント | 内容 |
|------------|------|
| [[detail_tsr]] | TSR：History Reprojection + Update + Upscale + Nyquist Filter |
| [[detail_taa]] | TAA：ジッタリング + History ブレンド + ゴースト抑制 |
| [[detail_bloom]] | Bloom：Downsample / Upsample / LensFlare / Dirt Mask 合成 |
| [[detail_tonemap]] | Tonemap：Exposure / Histogram / ACES / Gamma Correction |
| [[detail_dof]] | DOF：物理ベース CoC / Near＋Far Bokeh / Scatter-as-Gather |
| [[detail_motion_blur]] | MotionBlur：Velocity Tile / Dilation / Scatter-as-Gather |
