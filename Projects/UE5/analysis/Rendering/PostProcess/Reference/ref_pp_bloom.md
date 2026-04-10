# REF: Bloom 関連ファイル群

- 対象ファイル:
  - `PostProcessBloomSetup.h/.cpp`
  - `PostProcessFFTBloom.h/.cpp`
  - `PostProcessWeightedSampleSum.h/.cpp`
  - `PostProcessDownsample.h/.cpp`
- 関連Details: [[c_pp_bloom]]

---

## PostProcessBloomSetup.h/.cpp

### enum

```cpp
enum class EBloomQuality : int32
{
    Disabled = 0,
    Q1       = 1,  // 最低（2ダウンサンプル）
    Q2       = 2,
    Q3       = 3,
    Q4       = 4,
    Q5       = 5,  // 最高（5ダウンサンプル）
};
```

### 構造体

```cpp
struct FBloomSetupInputs
{
    FScreenPassTexture SceneColor;           // HDR SceneColor
    FRDGBufferRef EyeAdaptationBuffer;       // 露出値（輝度閾値に使用）
    float Threshold;                         // Bloom 発生輝度閾値
};
```

### 主要関数

```cpp
// Bloom の前処理（輝度閾値フィルタリング + ダウンサンプル）
FScreenPassTexture AddBloomSetupPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FBloomSetupInputs& Inputs);

// ガウシアン Bloom 全パス（複数回ダウンサンプル + ブラー + アップサンプル合成）
FScreenPassTexture AddGaussianBloomPasses(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FTextureDownsampleChain* DownsampleChain,
    const FScreenPassTexture& BloomSetup,
    EBloomQuality Quality);
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.BloomQuality` | 5 | Bloom 品質（0=無効, 1〜5） |
| `r.Bloom.Cross` | 0 | 十字形 Bloom 強度 |
| `r.Bloom.Threshold` | -1 | 輝度閾値（-1=プロジェクト設定） |

---

## PostProcessFFTBloom.h/.cpp

### 構造体

```cpp
struct FFFTBloomOutput
{
    FRDGTextureRef BloomTexture;     // FFT Bloom 結果テクスチャ
    FVector2f KernelCenterUV;        // カーネル中心の UV 座標
    FVector2f KernelSupportScaleUV;  // カーネルサポート範囲の UV スケール
};
```

### 主要関数

```cpp
// FFT Bloom が有効か判定
bool IsFFTBloomEnabled(const FViewInfo& View);

// FFT Bloom パスを追加
FFFTBloomOutput AddFFTBloomPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FScreenPassTexture& BloomSetupTexture,
    float InputResolutionFraction);
// InputResolutionFraction: 入力解像度の割合（FFT コスト削減のため低解像度で処理）
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.BloomMethod` | 0 | Bloom 方式（0=ガウシアン, 1=FFT） |
| `r.FFTBloom.KernelDownscale` | 1.0 | FFT カーネルのダウンスケール率 |
| `r.FFTBloom.Convolution.ImageTextureMaxResolutionFraction` | 1.0 | テクスチャ最大解像度率 |
| `r.FFTBloom.Convolution.CrossBlendWidth` | 0.1 | クロスブレンド幅 |
| `r.FFTBloom.Convolution.MinCrossBlendThreshold` | 1.0 | クロスブレンド最小閾値 |

---

## PostProcessWeightedSampleSum.h/.cpp

ガウシアンブラーの基盤。分離可能フィルター（水平 + 垂直の 2 パス）を実装する。

### 構造体

```cpp
struct FGaussianBlurInputs
{
    FScreenPassTexture Filter;         // ブラー対象テクスチャ
    FScreenPassTexture Additive;       // 加算合成テクスチャ（オプション）
    FLinearColor TintColor;            // ブラー色合い
    float CrossCenterWeight;           // 中心サンプルの追加重み
    float KernelSizePercent;           // カーネルサイズ（画面幅の%）
    bool bFastBlur;                    // 高速モード（サンプル数削減）
};
```

### 主要関数

```cpp
// ガウシアンブラー（H + V の 2 パス）
FScreenPassTexture AddGaussianBlurPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const TCHAR* Name,
    const FGaussianBlurInputs& Inputs);
```

---

## PostProcessDownsample.h/.cpp

### enum

```cpp
enum class EDownsampleQuality : uint8
{
    Low,   // 2×2 バイリニア（高速）
    High,  // 4×4 高品質
};

enum class EDownsampleFlags : uint8
{
    None         = 0,
    ForceRaster  = 1 << 0,  // Compute シェーダー使用不可時にラスタライズ強制
};
```

### 構造体

```cpp
struct FDownsamplePassInputs
{
    const TCHAR* Name = nullptr;
    FScreenPassTexture SceneColor;
    EDownsampleQuality Quality = EDownsampleQuality::Low;
    EDownsampleFlags Flags = EDownsampleFlags::None;
    EPixelFormat OverrideFormat = PF_Unknown;  // フォーマット指定（PF_Unknown=自動）
};

// マルチステージダウンサンプルチェーン
class FTextureDownsampleChain
{
public:
    static const uint32 StageCount = 5;  // 最大ステージ数（1/2, 1/4, 1/8, 1/16, 1/32）

    // 指定ステージが有効か
    bool IsStageEnabled(uint32 StageIndex) const;

    // 指定ステージのテクスチャ取得
    FScreenPassTexture GetTexture(uint32 StageIndex) const;

    // チェーンを構築（AddDownsamplePass を内部で呼ぶ）
    void Init(
        FRDGBuilder& GraphBuilder,
        const FViewInfo& View,
        const FScreenPassTexture& FirstStageInput,
        const EDownsampleQuality Quality,
        const EDownsampleFlags Flags,
        uint32 NumStages);
};
```

### 主要関数

```cpp
// 単一ダウンサンプルパス追加
FScreenPassTexture AddDownsamplePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FDownsamplePassInputs& Inputs);
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Downsample.Quality` | 1 | ダウンサンプル品質（0=Low, 1=High） |
