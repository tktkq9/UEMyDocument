# REF: プラットフォーム系ファイル群

- 対象ファイル（7本）:
  - `PostProcessMobile.h/.cpp`
  - `PostProcessAmbientOcclusionMobile.h/.cpp`
  - `PostProcessHMD.cpp`
  - `PostProcessDeviceEncodingOnly.h/.cpp`
  - `PostProcessMitchellNetravali.h/.cpp`（アップスケール用）
  - `SceneRenderTargets.h`（ヘッダのみ）
  - `PostProcessHMD.cpp`
- 関連Details: [[g_pp_misc]]

---

## 概要

特定のプラットフォーム・デバイス向けの PostProcess ファイル群。  
モバイル / VR / HDR 専用出力等、PC 向けパイプラインとは別の処理を提供する。

---

## PostProcessMobile.h/.cpp

### 役割

**モバイルプラットフォーム（iOS / Android 等）向けの軽量 PostProcess パイプライン**。  
`AddMobilePostProcessingPasses()` から呼ばれる。

PC 向けパイプラインとの主な違い:
- TAA の代わりに FXAA を使用
- DOF は簡易実装（PostProcessDOF.cpp の旧実装）
- Bloom は簡易ガウシアンのみ（FFT Bloom なし）
- ローカル露出処理なし

```cpp
// モバイル向け Post Process Material パス
FScreenPassTexture AddMobilePostProcessMaterialsPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    const FMobilePostProcessingInputs& Inputs);

// モバイル向けトーンマップ（シンプル実装）
FScreenPassTexture AddMobileTonemapperPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FTonemapInputs& Inputs);
```

### モバイル専用 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.MobileHDR` | 1 | モバイル HDR レンダリング |
| `r.MobileNumDynamicPointLights` | 4 | モバイルダイナミックポイントライト数上限 |
| `r.Mobile.EnableMovableSpotlights` | 0 | モバイルムーバブルスポットライト |
| `r.Mobile.PostProcessing.BloomType` | 0 | Bloom タイプ（0=Gaussian） |

---

## PostProcessAmbientOcclusionMobile.h/.cpp

### 役割

**モバイル向けの簡易アンビエントオクルージョン（SSAO）**。  
PC 向けの SSAO より低精度・低コストな実装。

```cpp
// モバイル AO が有効か
bool IsMobileAmbientOcclusionEnabled(const FViewInfo& View);

// モバイル AO パスを追加
FScreenPassTexture AddMobileAmbientOcclusionPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    FRDGTextureRef SceneDepth);
```

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Mobile.AmbientOcclusion` | 0 | モバイル AO 有効 |
| `r.Mobile.AmbientOcclusionQuality` | 1 | モバイル AO 品質 |

---

## PostProcessHMD.cpp

### 役割

**VR / HMD（Head-Mounted Display）向けのレンズディストーション補正**。  
HMD ごとの光学系歪みを補正してレンズ越しに正しく見えるようにする。

```cpp
// HMD レンズディストーション補正パス
FScreenPassTexture AddHMDDistortionPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor);
// IHeadMountedDisplay::GetDistortionParameters() から歪みマップを取得して適用
```

---

## PostProcessDeviceEncodingOnly.h/.cpp

### 役割

**デバイスエンコードのみを行う軽量 Tonemap パス**。  
フル PostProcess を無効にした場合でも最低限の色変換（ガンマ補正等）を行う。

```cpp
// デバイスエンコードのみ実行（Tonemap スキップ）
FScreenPassTexture AddDeviceEncodingOnlyPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FDeviceEncodingOnlyInputs& Inputs);

struct FDeviceEncodingOnlyInputs
{
    FScreenPassTexture SceneColor;
    FScreenPassRenderTarget Output;
    bool bFlipYAxis;
    bool bMetalMSAAHDRDecode;
};
```

---

## PostProcessMitchellNetravali.h/.cpp

### 役割

**Mitchell-Netravali バイキュービックフィルター**によるアップスケール実装。  
`r.Upscale.Quality=3` 相当の高品質リサンプルに使用される。

パラメータ B, C によってフィルターの特性が変化:
- B=1/3, C=1/3: Mitchell-Netravali （シャープ + リンギングバランス）
- B=0, C=0.5: Catmull-Rom（シャープ・リンギング多め）
- B=1, C=0: B-Spline（ソフト・リンギングなし）

```cpp
// Mitchell-Netravali アップスケールパスを追加
FScreenPassTexture AddMitchellNetravaliUpscalePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input,
    FIntPoint OutputSize,
    float B = 1.0f / 3.0f,
    float C = 1.0f / 3.0f);
```

---

## SceneRenderTargets.h

### 役割

PostProcess 系パスが参照するシーンレンダーターゲットの**型定義ヘッダ**。  
実装は `SceneRenderTargets.cpp`（PostProcess フォルダ外）にある。

```cpp
// PostProcess 側が参照する主要レンダーターゲット種別（例）
// ESceneColorFormatType, ECustomDepthPassLocation 等の定義
// 実体はレンダラーのコアモジュールが保持
```

---

## プラットフォーム分岐サマリー

| プラットフォーム | 特徴 |
|--------------|------|
| PC（デフォルト） | フルパイプライン（TAA/TSR, DiaphragmDOF, LocalExposure 等）|
| Mobile | 軽量パイプライン（FXAA, 簡易DOF, Gaussian Bloom のみ）|
| VR / HMD | HMD レンズディストーション補正を追加 |
| コンソール | プラットフォーム固有最適化（RHI レイヤーで対応）|
| MinimalHDR | DeviceEncodingOnly でガンマ変換のみ |
