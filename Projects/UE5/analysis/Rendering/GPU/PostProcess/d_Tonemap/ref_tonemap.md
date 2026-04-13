# リファレンス：PostProcessTonemap / PostProcessEyeAdaptation / PostProcessHistogram エントリポイント

- グループ: d - Tonemap
- 上位: [[detail_tonemap]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessTonemap.h/.cpp`
  - `Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessEyeAdaptation.h/.cpp`
  - `Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessHistogram.h/.cpp`
  - `Engine/Source/Runtime/Renderer/Private/PostProcess/ACESUtils.h/.cpp`

## 主要関数

### PostProcessHistogram.h

| 関数 | 説明 |
|------|------|
| `AddHistogramPass(...)` | 輝度ヒストグラム（256バケット）を生成 |
| `AddLocalExposurePass(...)` | Local Exposure Bilateral Grid を生成 |

### PostProcessEyeAdaptation.h

| 関数 | 説明 |
|------|------|
| `AddHistogramEyeAdaptationPass(...)` | ヒストグラム → 露出値バッファ更新 |
| `AddBasicEyeAdaptationPass(...)` | Basic（非ヒストグラム）露出値更新 |
| `AddBasicEyeAdaptationSetupPass(...)` | SceneColor の Luma を Alpha に格納 |
| `GetEyeAdaptationParameters(View)` | `FEyeAdaptationParameters` を View から取得 |
| `GetAutoExposureMethod(View)` | 露出方式（Histogram / Basic / Manual / Fixed）を返す |
| `IsAutoExposureMethodSupported(FeatureLevel, Method)` | FeatureLevel でサポートされるか |
| `IsExtendLuminanceRangeEnabled()` | 拡張輝度範囲が有効か |

### PostProcessTonemap.h

| 関数 | 説明 |
|------|------|
| `AddTonemapPass(GraphBuilder, View, Inputs)` | Tonemap の RDG パスを追加 |
| `GetTonemapperOutputDeviceParameters(Family)` | 出力デバイスパラメータを取得 |
| `SupportsFilmGrain(Platform)` | Film Grain サポートチェック |
| `FTonemapInputs::SupportsSceneColorApplyParametersBuffer(Platform)` | SceneColorApplyParameters バッファ対応確認 |

---

## `FEyeAdaptationParameters` 全メンバ

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `ExposureLowPercent` | `float` | ヒストグラム低端カットオフ（%） |
| `ExposureHighPercent` | `float` | ヒストグラム高端カットオフ（%） |
| `MinAverageLuminance` | `float` | 平均輝度クランプ下限 |
| `MaxAverageLuminance` | `float` | 平均輝度クランプ上限 |
| `ExposureCompensationSettings` | `float` | EV 露出補正（設定値） |
| `ExposureCompensationCurve` | `float` | EV 露出補正（カーブ） |
| `DeltaWorldTime` | `float` | フレームデルタ時間 |
| `ExposureSpeedUp` | `float` | 明るくなる速度 |
| `ExposureSpeedDown` | `float` | 暗くなる速度 |
| `HistogramScale` / `HistogramBias` | `float` | 輝度→ヒストグラムインデックス変換 |
| `LuminanceMin` | `float` | 有効輝度の下限 |
| `LuminanceMax` | `float` | センサー飽和輝度 |
| `LuminanceWeights` | `FVector3f` | RGB 輝度重み（BT.709） |
| `ForceTarget` | `float` | 強制露出値（デバッグ） |

---

## `FTonemapperOutputDeviceParameters` メンバ

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `InverseGamma` | `FVector3f` | 出力ガンマの逆数（チャンネルごと） |
| `OutputDevice` | `uint32` | 出力デバイス種別（SDR=0, HDR10=1, ...） |
| `OutputGamut` | `uint32` | 色域（Rec.709=0, P3=1, Rec.2020=2, ...） |
| `OutputMaxLuminance` | `float` | HDR ピーク輝度（nit） |

---

## `AutoExposurePermutation` 名前空間

```cpp
namespace AutoExposurePermutation
{
    class FUsePrecalculatedLuminanceDim : SHADER_PERMUTATION_BOOL("USE_PRECALCULATED_LUMINANCE");
    class FUseApproxIlluminanceDim : SHADER_PERMUTATION_BOOL("USE_APPROX_ILLUMINANCE");
    class FUseDebugOutputDim : SHADER_PERMUTATION_BOOL("USE_DEBUG_OUTPUT");

    using FCommonDomain = TShaderPermutationDomain<
        FUsePrecalculatedLuminanceDim,
        FUseApproxIlluminanceDim,
        FUseDebugOutputDim>;
}
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.EyeAdaptationQuality` | 2 | 0=無効, 1=Basic, 2=Histogram |
| `r.Tonemapper.Quality` | 5 | トーンマップ品質パーミュテーション |
| `r.HDR.EnableHDROutput` | 0 | HDR 出力有効化 |
| `r.LocalExposure.HighlightContrastScale` | 0.8 | Local Exposure ハイライト強度 |
| `r.LocalExposure.ShadowContrastScale` | 0.8 | Local Exposure シャドウ強度 |
