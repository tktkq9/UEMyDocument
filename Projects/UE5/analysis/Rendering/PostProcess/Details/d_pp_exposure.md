# PostProcess d: 露出管理（Histogram / EyeAdaptation / LocalExposure）

- 対象: `PostProcess/PostProcessHistogram.h/.cpp` / `PostProcessEyeAdaptation.h/.cpp` / `PostProcessLocalExposure.h/.cpp`
- 上位: [[05_postprocess_overview]]

---

## 役割

シーン全体・局所的な輝度を計算し、カメラの**自動露出**と**ローカル露出補正**を実現する。

```
Histogram → EyeAdaptation → 露出値を Tonemap に渡す
                                    ↓
LocalExposure → Tonemapでの色調補正に使用
```

---

## 1. PostProcessHistogram（輝度ヒストグラム）

### 役割

SceneColor の輝度分布を GPU 上でヒストグラム集計する。  
EyeAdaptation の入力データを提供する。

### 処理フロー

```
[1] AddHistogramPass()
    → SceneColor を Compute シェーダーで走査
    → 各ピクセルの log2 輝度を HistogramBuffer（64〜256 ビン）に累積

[2] 出力: FRDGBufferRef HistogramBuffer
    → EyeAdaptation に渡す
```

```cpp
FRDGBufferRef AddHistogramPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FEyeAdaptationParameters& EyeAdaptationParameters,
    FScreenPassTexture SceneColor,
    const FSceneTextureParameters& SceneTextures,
    FRDGBufferRef LastEyeAdaptationBuffer);
```

---

## 2. PostProcessEyeAdaptation（自動露出）

### 役割

ヒストグラムから露出値を計算し、**フレーム間でなめらかに適応**させる。  
結果は `EyeAdaptationBuffer` に書き込まれ Tonemap で使用される。

### 3つの実装方式

| 方式 | 関数 | 説明 |
|-----|-----|------|
| ヒストグラムベース | `AddHistogramEyeAdaptationPass()` | 精度高・デフォルト |
| 基本（ルマ平均） | `AddBasicEyeAdaptationPass()` | 軽量・モバイル向け |
| 手動露出 | - | `r.EyeAdaptation.MethodOverride=-1` 以外を設定 |

### FEyeAdaptationParameters

```cpp
struct FEyeAdaptationParameters
{
    // 露出範囲
    float ExposureLowPercent;       // ヒストグラムの下位%をカット
    float ExposureHighPercent;      // ヒストグラムの上位%をカット

    // 輝度の対数範囲
    float HistogramLogMin;
    float HistogramLogMax;

    // 適応速度
    float ExposureSpeedUp;          // 明るい環境への適応速度
    float ExposureSpeedDown;        // 暗い環境への適応速度

    // 補正値
    float ExposureCompensation;     // 露出補正（EV）
    float ExposureCalibrationConstant;

    // 最小/最大露出
    float MinAverageLuminance;
    float MaxAverageLuminance;

    // メータリングモード
    uint32 MeterMask;               // スポット / センター重点 / 評価測光
};
```

### 主要関数

```cpp
// パラメータを View から取得
FEyeAdaptationParameters GetEyeAdaptationParameters(
    const FViewInfo& View,
    ERHIFeatureLevel::Type MinFeatureLevel);

// ヒストグラムベース露出適応パス
FRDGBufferRef AddHistogramEyeAdaptationPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FEyeAdaptationParameters& EyeAdaptationParameters,
    FRDGBufferRef HistogramBuffer);

// 基本露出適応パス（軽量）
FRDGBufferRef AddBasicEyeAdaptationPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FEyeAdaptationParameters& EyeAdaptationParameters,
    FScreenPassTexture SceneColor,
    FRDGBufferRef LastEyeAdaptationBuffer);

// 有効化チェック
bool IsAutoExposureMethodSupported(EAutoExposureMethod Method, ERHIFeatureLevel::Type FeatureLevel);
```

---

## 3. PostProcessLocalExposure（ローカル露出）

### 役割

**画面内の領域ごとに異なる露出補正**を適用する。  
ハイライトが飛ぶ問題や黒つぶれを局所的に緩和する。

### 処理フロー

```
[1] AddLocalExposureBlurredLogLuminancePass()
    → SceneColor の log 輝度マップを生成
    → ガウシアンブラーで低周波輝度マップを作成（FTextureDownsampleChain 使用）

[2] AddApplyLocalExposurePass()
    → 各ピクセルのローカル輝度 vs グローバル輝度の比でスケール計算
    → HighlightContrastScale / ShadowContrastScale で調整

[3] AddLocalExposureFusionPass()  ← 内部的に AddApplyLocalExposurePass() に統合
    → HDR テクスチャとローカル露出補正結果を合成
```

### FLocalExposureParameters

```cpp
struct FLocalExposureParameters
{
    float HighlightContrastScale;   // ハイライトのコントラスト抑制（0〜1）
    float ShadowContrastScale;      // シャドウのコントラスト強調
    float DetailStrength;           // ディテール強度
    float BlurredLuminanceBlend;    // ブラー輝度との混合率
    float MiddleGreyExposureCompensation; // 中間グレー露出補正
};
```

### 主要関数

```cpp
// log 輝度ブラーパスを追加
FRDGTextureRef AddLocalExposureBlurredLogLuminancePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FEyeAdaptationParameters& EyeAdaptationParameters,
    FScreenPassTexture SceneColor);

// ローカル露出適用パス
FScreenPassTexture AddLocalExposurePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    FRDGTextureRef BlurredLogLuminanceTexture,
    FRDGBufferRef EyeAdaptationBuffer,
    const FLocalExposureParameters& LocalExposureParameters);
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.EyeAdaptation.MethodOverride` | -1 | 露出手法オーバーライド（-1=自動選択） |
| `r.EyeAdaptation.ExposureCompensationCurve` | 1 | 露出補正カーブ使用 |
| `r.EyeAdaptation.LuminancePriority` | 1 | 輝度優先測光 |
| `r.EyeAdaptation.IgnoreMaterials` | 0 | マテリアル依存の露出無視 |
| `r.LocalExposure.HighlightContrastScale` | 0.8 | ハイライトコントラスト |
| `r.LocalExposure.ShadowContrastScale` | 0.8 | シャドウコントラスト |
| `r.LocalExposure.DetailStrength` | 1.0 | ディテール強度 |
| `r.LocalExposure.BlurredLuminanceBlend` | 1.0 | ブラー輝度混合率 |
| `r.HistogramReduce` | 1 | ヒストグラム縮小（パフォーマンス） |
