# リファレンス：PostProcessBloomSetup.h / PostProcessLensFlares.h エントリポイント

- グループ: c - Bloom
- 上位: [[detail_bloom]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessBloomSetup.h/.cpp`
  - `Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessLensFlares.h/.cpp`
  - `Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessFFTBloom.h/.cpp`

## 主要関数・型

### `EBloomQuality`

```cpp
enum class EBloomQuality : uint32
{
    Disabled, Q1, Q2, Q3, Q4, Q5, MAX
};
EBloomQuality GetBloomQuality(); // r.Bloom.Quality CVar から取得
```

### `AddBloomSetupPass`

```cpp
FScreenPassTexture AddBloomSetupPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FBloomSetupInputs& Inputs);
```

Bloom 入力テクスチャを生成する。`Inputs.Threshold` を超える輝度ピクセルを抽出する。

### `AddGaussianBloomPasses`

```cpp
FScreenPassTexture AddGaussianBloomPasses(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FTextureDownsampleChain* SceneDownsampleChain);
```

Gaussian Bloom の Downsample → Upsample チェーンを構築して最終 Bloom テクスチャを返す。

### `AddLensFlarePass`

```cpp
FScreenPassTexture AddLensFlarePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Bloom);
```

Bloom テクスチャを基に Lens Flare と Dirt Mask を合成して返す。

---

## `FBloomSetupInputs` メンバ

| 変数名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| `SceneColor` | `FScreenPassTextureSlice` | ✅ | HDR シーンカラー |
| `EyeAdaptationBuffer` | `FRDGBufferRef` | ✅ | 露出値バッファ |
| `Threshold` | `float` | ✅ | 輝度閾値（> 0） |
| `EyeAdaptationParameters` | `const FEyeAdaptationParameters*` | ― | 露出パラメータ |
| `LocalExposureParameters` | `const FLocalExposureParameters*` | ― | ローカル露出 |
| `LocalExposureTexture` | `FRDGTextureRef` | ― | ローカル露出テクスチャ |
| `BlurredLogLuminanceTexture` | `FRDGTextureRef` | ― | ぼかし済み輝度 |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Bloom.Quality` | 5 | 0=無効, 1〜5=品質 |
| `r.BloomFFT` | 0 | FFT Bloom 有効化 |
| `r.BloomThreshold` | -1 | 輝度閾値（-1=PV 依存） |
| `r.BloomDirtMask` | 1 | レンズ汚れマスク |
| `r.LensFlare` | 1 | Lens Flare 有効化 |
