# REF: Material / Subsurface / Misc ファイル群

- 対象ファイル:
  - `PostProcessMaterial.h/.cpp`
  - `PostProcessSubsurface.h/.cpp`
  - `PostProcessLensFlares.h/.cpp`
  - `PostProcessUpscale.h/.cpp`
  - `NeuralPostProcess.h/.cpp`
  - `PostProcessAA.cpp`
  - `SubpixelMorphologicalAA.cpp`
  - `LensDistortion.cpp`
  - `PostProcessMitchellNetravali.h/.cpp`
  - `AlphaInvert.h/.cpp`
  - `SceneFilterRendering.h/.cpp`
- 関連Details: [[g_pp_misc]]

---

## PostProcessMaterial.h/.cpp

### 構造体

```cpp
// ポストプロセスマテリアルへのシーンテクスチャ入力
struct FPostProcessMaterialParameters
{
    FRDGTextureRef SceneColor;
    FRDGTextureRef SceneDepth;
    FRDGTextureRef SceneNormal;
    FRDGTextureRef SceneVelocity;
    FRDGTextureRef SeparateTranslucency;
    // GBuffer A/B/C/D, CustomDepth, CustomStencil 等も含む
};

// スクリーンショット用マスク入力
struct FHighResolutionScreenshotMaskInputs
{
    FScreenPassTexture SceneColor;
    FScreenPassTexture Mask;
    UMaterialInterface* MaskMaterial;
};
```

### 主要関数

```cpp
// 単一マテリアルのポストプロセスパス追加
FScreenPassTexture AddPostProcessMaterialPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FPostProcessMaterialParameters& Parameters,
    const UMaterialInterface* Material,
    FScreenPassTexture Input);

// 指定ブレンダブルロケーションのマテリアルチェーンを一括追加
FScreenPassTexture AddPostProcessMaterialChain(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FPostProcessMaterialParameters& Parameters,
    FScreenPassTexture Input,
    EBlendableLocation Location,
    const TArray<UMaterialInterface*>& Materials);

// スクリーンショット用マスクパス
FScreenPassTexture AddHighResolutionScreenshotMaskPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FHighResolutionScreenshotMaskInputs& Inputs);
```

---

## PostProcessSubsurface.h/.cpp

### 主要関数

```cpp
// SSS が有効か（GBuffer に SSS データが含まれるか）
bool IsSubsurfaceEnabled(const FViewInfo& View);

// SSS 後処理パスを追加（GBuffer の SSS マスクをブラーして合成）
FScreenPassTexture AddSubsurfacePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor);

// SSS 可視化デバッグパス
FScreenPassTexture AddVisualizeSubsurfacePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor);
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.SSS.Quality` | 1 | SSS 品質（0=無効, 1=Burley, 2=高品質） |
| `r.SSS.Filter` | 1 | SSS フィルタータイプ |
| `r.SSS.HalfRes` | 1 | ハーフ解像度で処理 |
| `r.SSS.Scale` | 1.0 | SSS スケール |

---

## PostProcessLensFlares.h/.cpp

### 構造体・enum

```cpp
enum class ELensFlareQuality : uint8
{
    Disabled = 0,
    Low      = 1,
    High     = 2,
    VeryHigh = 3,
};

struct FLensFlareInputs
{
    FScreenPassTexture Bloom;                 // Bloom テクスチャ（フレアのソース輝度）
    FRDGTextureRef BokehShapeTexture;         // ボケ形状テクスチャ（オプション）
    FLinearColor TintColors[8];               // ゴーストごとの色合い（最大8個）
    float Intensity;                          // フレア強度
    float Threshold;                          // 発生輝度閾値
    float BokehSize;                          // ボケサイズ
    bool bCompositeWithBloom;                 // Bloom に合成するか
};
```

### 主要関数

```cpp
FScreenPassTexture AddLensFlaresPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Bloom,
    const FLensFlareInputs& Inputs);

bool IsLensFlareEnabled(const FViewInfo& View);
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.LensFlareQuality` | 2 | レンズフレア品質 |

---

## PostProcessUpscale.h/.cpp

### 構造体・enum

```cpp
enum class EUpscaleMethod : uint8
{
    None, Nearest, Bilinear, Bicubic,
    BicubicSharper, CatmullRom, Lanczos,
    Gaussian, SmoothStep, Temporal, TSR,
};

enum class EUpscaleStage : uint8
{
    PrimaryToSecondary,   // 一次解像度 → 二次解像度
    PrimaryToOutput,      // 一次解像度 → 最終出力
    SecondaryToOutput,    // 二次解像度 → 最終出力
};

// カスタムアップスケーラーのプラグインインターフェース（DLSS / XeSS 等）
class ISpatialUpscaler
{
public:
    struct FInputs
    {
        FScreenPassTexture InputTexture;
        EUpscaleStage Stage;
        EUpscaleMethod Method;
    };

    virtual FScreenPassTexture AddPasses(
        FRDGBuilder& GraphBuilder,
        const FViewInfo& View,
        const FInputs& Inputs) const = 0;

    virtual const TCHAR* GetDebugName() const = 0;

    // ゲームスレッドでコピーを作成（マルチビュー対応）
    virtual ISpatialUpscaler* Fork_GameThread(const FSceneViewFamily&) const = 0;
};
```

### 主要関数

```cpp
FScreenPassTexture AddSpatialUpscalePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    EUpscaleStage Stage,
    EUpscaleMethod Method,
    FScreenPassTexture Input);
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Upscale.Quality` | 3 | アップスケール品質 |
| `r.Upscale.Softness` | 1.0 | ソフトネス |
| `r.Upscale.PaniniProjection` | 0 | パニーニ投影 |

---

## NeuralPostProcess.h/.cpp

```cpp
struct FNeuralPostProcessResource
{
    TSharedPtr<UE::NNE::IModelInstanceGPU> ModelInstance;
    TArray<FRDGTextureRef> IntermediateTextures;
};

void AllocateNeuralPostProcessingResourcesIfNeeded(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FNeuralPostProcessResource& OutResource);

FScreenPassTexture ApplyNeuralPostProcess(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input,
    FNeuralPostProcessResource& Resource);
```

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.NeuralPostProcess.Enable` | 0 | ニューラル後処理有効（実験的）|

---

## その他ユーティリティ

### PostProcessAA.cpp（FXAA）

```cpp
// FXAA（Fast Approximate Anti-Aliasing）パス追加
FScreenPassTexture AddPostProcessAA(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input);
```

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.FXAA.Quality` | 3 | FXAA 品質（0〜5）|

### SubpixelMorphologicalAA.cpp（SMAA）

```cpp
// SMAA（Subpixel Morphological Anti-Aliasing）パス追加
FScreenPassTexture AddSMAAPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input);
```

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.SMAA.Quality` | 1 | SMAA 品質 |

### PostProcessMitchellNetravali.h/.cpp

Mitchell-Netravali フィルターを使った高品質リサンプル（アップスケール用）。

```cpp
FScreenPassTexture AddMitchellNetravaliUpscalePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input,
    FIntPoint OutputSize);
```

### SceneFilterRendering.h/.cpp

フルスクリーンパスのユーティリティ。

```cpp
// フルスクリーン矩形を描画（UV 座標付き）
void DrawRectangle(
    FRHICommandList& RHICmdList,
    float X, float Y, float SizeX, float SizeY,
    float U, float V, float SizeU, float SizeV,
    FIntPoint TargetSize, FIntPoint TextureSize,
    const TShaderRef<FShader>& VertexShader,
    EDrawRectangleFlags Flags = EDRF_Default);
```

### AlphaInvert.h/.cpp

```cpp
// アルファチャンネルを反転（1 - Alpha）
FScreenPassTexture AddAlphaInvertPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input);
```

### LensDistortion.cpp

```cpp
// レンズディストーション補正パス（VR・広角レンズ向け）
FScreenPassTexture AddLensDistortionPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input);
```

---

> [!note]- ISpatialUpscaler によるプラグイン差し替えアーキテクチャ
> `ISpatialUpscaler` は DLSS・XeSS・FSR 等のサードパーティアップスケーラーをプラグインとして差し込むための抽象インターフェース。  
> `Fork_GameThread()` でゲームスレッド側にコピーを作り、マルチビュー（VR 等）でも各ビューに独立したインスタンスを渡せる設計になっている。  
> プラグインが登録されていない場合は `AddSpatialUpscalePass()` がデフォルトの Bicubic / CatmullRom フィルターにフォールバックする。

> [!note]- PostProcessMaterial の EBlendableLocation と挿入タイミング
> `AddPostProcessMaterialChain()` は `EBlendableLocation` を受け取り、PP パイプライン内の挿入ポジション（`BL_AfterOpaque`・`BL_BeforeTranslucency`・`BL_AfterTranslucency`・`BL_BeforeTonemap`・`BL_AfterTonemapping`）を決定する。  
> `BL_BeforeTonemap` はトーンマップ前の HDR 空間でマテリアルを適用するため、線形 RGB 計算が必要。`BL_AfterTonemapping` は LDR（sRGB）空間で動作する。  
> `AddPostProcessingPasses()` がロケーションごとにこの関数を呼び出すことで、マテリアルチェーンがパイプラインの適切な位置に挿入される。

> [!note]- SSS の Burley と画面空間フィルタリング
> `AddSubsurfacePass()` は GBuffer の SSS プロファイルマスクを使って、肌・蝋・牛乳等の素材に散乱を適用する。  
> `r.SSS.Quality = 1`（Burley）は物理ベースの Burley 正規化拡散プロファイルをスクリーンスペースで近似し、HalfRes で処理後フル解像度に合成する（`r.SSS.HalfRes = 1`）。  
> `r.SSS.Quality = 0` では SSS が完全にスキップされ、散乱なしの直接ライティングのみが適用される。
