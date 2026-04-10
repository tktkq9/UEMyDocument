# PostProcess c: Bloom

- 対象: `PostProcess/PostProcessBloomSetup.h/.cpp` / `PostProcessFFTBloom.h/.cpp` / `PostProcessWeightedSampleSum.h/.cpp` / `PostProcessDownsample.h/.cpp`
- 上位: [[05_postprocess_overview]]

---

## 役割

輝度の高い部分からにじみ出るグロー効果（Bloom）を生成する。  
**2つの実装**を `r.BloomMethod` で切り替え:

| 実装 | ファイル | 特徴 |
|-----|---------|------|
| ガウシアン Bloom | BloomSetup + WeightedSampleSum | 高速・複数パス分離 |
| FFT Bloom | PostProcessFFTBloom | 高品質・任意カーネル形状 |

---

## ガウシアン Bloom パイプライン

```
[1] AddBloomSetupPass()
    → SceneColor から Bloom 対象ピクセルを抽出（輝度閾値でフィルタリング）
    → FBloomSetupInputs: SceneColor + EyeAdaptation + Threshold

[2] AddGaussianBloomPasses()
    → 複数回の AddGaussianBlurPass() でブラー

         DownsampleChain
         SceneColor → 1/2 → 1/4 → 1/8 → 1/16 → 1/32
                       ↓      ↓      ↓      ↓      ↓
              BloomDown1 BloomDown2 BloomDown3 BloomDown4 BloomDown5

         各ダウンサンプルに Horizontal + Vertical の2パスガウシアンブラー
         その後 Upsample して加算合成

[3] 出力: BloomTexture（SceneColor に後でAddされる）
```

### EBloomQuality

```cpp
enum class EBloomQuality : int32
{
    Disabled = 0, // Bloom なし
    Q1       = 1, // 最低品質（2ダウンサンプルのみ）
    Q2       = 2,
    Q3       = 3,
    Q4       = 4,
    Q5       = 5, // 最高品質（5ダウンサンプル）
};
```

---

## FFT Bloom

```
[1] SceneColor → ダウンサンプル（解像度削減でFFTコスト削減）
[2] FFT変換（空間 → 周波数領域）
[3] カーネルとの乗算（畳み込み = ブルーム形状の決定）
    → テクスチャで任意の Bloom 形状を指定可能
[4] 逆FFT → ブルーム結果
[5] SceneColor に合成
```

### 主要関数

```cpp
// FFT Bloom が有効かどうか判定
bool IsFFTBloomEnabled(const FViewInfo& View);

// FFT Bloom パスを追加
FFFTBloomOutput AddFFTBloomPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FScreenPassTexture& BloomSetupTexture,
    float InputResolutionFraction);

struct FFFTBloomOutput
{
    FRDGTextureRef BloomTexture;    // Bloom 結果テクスチャ
    FVector2f KernelCenterUV;       // カーネル中心 UV
    FVector2f KernelSupportScaleUV; // カーネルサポート範囲
};
```

---

## WeightedSampleSum（ガウシアンブラー基盤）

```cpp
struct FGaussianBlurInputs
{
    FScreenPassTexture Filter;     // ブラー対象テクスチャ
    FScreenPassTexture Additive;   // 加算合成テクスチャ（オプション）
    FLinearColor TintColor;        // 色合い
    float CrossCenterWeight;       // 十字方向の重み
    float KernelSizePercent;       // カーネルサイズ（画面の%）
    bool bFastBlur;                // 高速ブラーモード
};

// ガウシアンブラーパスを追加（H方向 + V方向の2パス）
FScreenPassTexture AddGaussianBlurPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const TCHAR* Name,
    const FGaussianBlurInputs& Inputs);
```

---

## Downsample

```cpp
enum class EDownsampleQuality : uint8
{
    Low,  // 2×2（バイリニア）
    High, // 4×4（高品質）
};

struct FDownsamplePassInputs
{
    const TCHAR* Name;
    FScreenPassTexture SceneColor;
    EDownsampleQuality Quality;
    EDownsampleFlags Flags;
    EPixelFormat OverrideFormat;
};

// 単一ダウンサンプルパス
FScreenPassTexture AddDownsamplePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FDownsamplePassInputs& Inputs);

// マルチステージダウンサンプルチェーン管理
class FTextureDownsampleChain
{
    // ステージ数（例: 1/2, 1/4, 1/8, 1/16, 1/32）
    static const uint32 StageCount = 5;
    TArray<FScreenPassTexture> Textures;

    bool IsStageEnabled(uint32 StageIndex) const;
    FScreenPassTexture GetTexture(uint32 StageIndex) const;
};
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.BloomQuality` | 5 | Bloom 品質（0=無効, 1〜5） |
| `r.BloomMethod` | 0 | Bloom方式（0=ガウシアン, 1=FFT） |
| `r.Bloom.Cross` | 0 | 十字形状 Bloom 強度 |
| `r.FFTBloom.KernelDownscale` | 1.0 | FFT カーネルダウンスケール率 |
| `r.FFTBloom.Convolution.ImageTextureMaxResolutionFraction` | 1.0 | FFT テクスチャ最大解像度率 |
| `r.Downsample.Quality` | 1 | ダウンサンプル品質（0=Low, 1=High） |
