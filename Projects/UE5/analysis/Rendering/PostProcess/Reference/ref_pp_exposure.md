# REF: 露出管理ファイル群

- 対象ファイル:
  - `PostProcessHistogram.h/.cpp`
  - `PostProcessEyeAdaptation.h/.cpp`
  - `PostProcessLocalExposure.h/.cpp`
- 関連Details: [[d_pp_exposure]]

---

## PostProcessHistogram.h/.cpp

### 役割

SceneColor の輝度を GPU 上でヒストグラム集計する。出力は EyeAdaptation の入力。

### 主要関数

```cpp
// ヒストグラム計算パスを追加
FRDGBufferRef AddHistogramPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FEyeAdaptationParameters& EyeAdaptationParameters,
    FScreenPassTexture SceneColor,
    const FSceneTextureParameters& SceneTextures,
    FRDGBufferRef LastEyeAdaptationBuffer);
// 戻り値: uint32[] の HistogramBuffer（64 or 256 ビン）
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.HistogramReduce` | 1 | ヒストグラム縮小パス使用 |
| `r.Histogram.NumBilinearWithMipMaps` | 0 | バイリニアMIPマップ使用 |

---

## PostProcessEyeAdaptation.h/.cpp

### FEyeAdaptationParameters

```cpp
struct FEyeAdaptationParameters
{
    // 輝度の使用範囲（外れ値を除外）
    float ExposureLowPercent;          // 下位%をカット（デフォルト: 80）
    float ExposureHighPercent;         // 上位%をカット（デフォルト: 98）

    // 対数輝度の範囲
    float HistogramLogMin;             // 最小 log 輝度（デフォルト: -8）
    float HistogramLogMax;             // 最大 log 輝度（デフォルト: 4）

    // 適応速度
    float ExposureSpeedUp;             // 明るい方向への適応速度（EV/秒）
    float ExposureSpeedDown;           // 暗い方向への適応速度（EV/秒）

    // 露出補正
    float ExposureCompensation;        // EV 補正値
    float ExposureCalibrationConstant; // キャリブレーション定数（デフォルト: 0.18）

    // 輝度クランプ
    float MinAverageLuminance;
    float MaxAverageLuminance;

    // デルタ時間
    float DeltaWorldTime;              // フレーム時間（適応速度計算用）

    // 測光マスク
    uint32 MeterMask;                  // スポット / センター重点 / 評価測光

    // 手動露出
    bool bExtendDefaultLuminanceRange;
    float FixedExposureBrightness;
};
```

### 主要関数

```cpp
// View から EyeAdaptation パラメータを取得
FEyeAdaptationParameters GetEyeAdaptationParameters(
    const FViewInfo& View,
    ERHIFeatureLevel::Type MinFeatureLevel);

// ヒストグラムベース露出適応パス（精度高・デフォルト）
FRDGBufferRef AddHistogramEyeAdaptationPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FEyeAdaptationParameters& EyeAdaptationParameters,
    FRDGBufferRef HistogramBuffer);

// 基本露出適応パス（ルマ平均・軽量）
FRDGBufferRef AddBasicEyeAdaptationPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FEyeAdaptationParameters& EyeAdaptationParameters,
    FScreenPassTexture SceneColor,
    FRDGBufferRef LastEyeAdaptationBuffer);

// 露出手法が使用可能か
bool IsAutoExposureMethodSupported(
    EAutoExposureMethod Method,
    ERHIFeatureLevel::Type FeatureLevel);

// 指定ビューが自動露出を使用しているか
bool IsExtendLuminanceRangeEnabled(const FSceneViewFamily& Family);
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.EyeAdaptation.MethodOverride` | -1 | 手法強制指定（-1=自動, 0=None, 1=Histogram, 2=Basic） |
| `r.EyeAdaptation.ExposureCompensationCurve` | 1 | 露出補正カーブ使用 |
| `r.EyeAdaptation.LuminancePriority` | 1 | 輝度優先測光 |
| `r.EyeAdaptation.IgnoreMaterials` | 0 | マテリアル依存の露出無視 |
| `r.EyeAdaptation.EditorOnly.IgnoreHMD` | 0 | エディタでHMD露出無視 |
| `r.EyeAdaptation.VisualizeDebugType` | 0 | デバッグ可視化タイプ |

---

## PostProcessLocalExposure.h/.cpp

### FLocalExposureParameters

```cpp
struct FLocalExposureParameters
{
    float HighlightContrastScale;          // ハイライトのコントラスト抑制（0〜1）
    float ShadowContrastScale;             // シャドウのコントラスト強調（0〜1）
    float DetailStrength;                  // ディテール強度
    float BlurredLuminanceBlend;           // ブラー輝度との混合率
    float MiddleGreyExposureCompensation;  // 中間グレー露出補正
};
```

### 主要関数

```cpp
// ローカル露出計算に必要な log 輝度ブラーパスを追加
FRDGTextureRef AddLocalExposureBlurredLogLuminancePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FEyeAdaptationParameters& EyeAdaptationParameters,
    FScreenPassTexture SceneColor);
// 戻り値: ぼかした log 輝度テクスチャ（FTextureDownsampleChain で生成）

// ローカル露出適用パス
FScreenPassTexture AddLocalExposurePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    FRDGTextureRef BlurredLogLuminanceTexture,
    FRDGBufferRef EyeAdaptationBuffer,
    const FLocalExposureParameters& LocalExposureParameters);

// ローカル露出が有効か
bool IsLocalExposureEnabled(const FViewInfo& View);
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.LocalExposure.HighlightContrastScale` | 0.8 | ハイライトコントラスト抑制 |
| `r.LocalExposure.ShadowContrastScale` | 0.8 | シャドウコントラスト強調 |
| `r.LocalExposure.DetailStrength` | 1.0 | ディテール強度 |
| `r.LocalExposure.BlurredLuminanceBlend` | 1.0 | ブラー輝度との混合率 |
| `r.LocalExposure.MiddleGreyExposureCompensation` | 0 | 中間グレー露出補正 |
| `r.LocalExposure.BilateralGridSize` | 16 | バイラテラルグリッドサイズ |

---

> [!note]- ヒストグラム集計の仕組みと 256 ビン
> `AddHistogramPass()` (`PostProcessHistogram.cpp:451`) は SceneColor の全ピクセルから log2 輝度を計算し、  
> 256 ビンのヒストグラムをアトミック加算 CS で構築する。  
> `r.EyeAdaptation.MinBrightness`〜`MaxBrightness` の log 範囲を 256 等分し、各ピクセルを該当ビンに加算する。  
> `HistogramReduce` でタイル単位に縮小することでアトミック競合を削減している。

> [!note]- EyeAdaptation の指数移動平均による応答速度
> `AddHistogramEyeAdaptationPass()` (`PostProcessEyeAdaptation.cpp:1023`) はヒストグラムから目標 EV を求め、  
> 前フレームの EV と指数移動平均でブレンドすることで応答の遅延をシミュレートする。  
> `ExposureSpeedUp`（明→暗応答）と `ExposureSpeedDown`（暗→明応答）を別々に設定でき、  
> 人間の目の非対称な光適応特性を再現している。

> [!note]- LocalExposure のバイラテラルグリッド
> `AddLocalExposureBlurredLogLuminancePass()` は log 空間での輝度をバイラテラルグリッドで平滑化する。  
> `r.LocalExposure.BilateralGridSize = 16` で空間解像度 16x16 のグリッドを作成し、  
> エッジを保存しながら低周波の輝度分布を推定する。これをもとにハイライトとシャドウを個別に圧縮・伸張する。
