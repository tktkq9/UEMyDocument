# PostProcess g: その他機能（Material / Subsurface / LensFlares / Upscale / Neural / AA）

- 対象: `PostProcessMaterial.h/.cpp` / `PostProcessSubsurface.h/.cpp` / `PostProcessLensFlares.h/.cpp` / `PostProcessUpscale.h/.cpp` / `NeuralPostProcess.h/.cpp` / `PostProcessAA.cpp` / `SubpixelMorphologicalAA.cpp` / `LensDistortion.cpp` / `PostProcessMitchellNetravali.h/.cpp` / `AlphaInvert.h/.cpp` / `SceneFilterRendering.h/.cpp`
- 上位: [[05_postprocess_overview]]

---

## 1. PostProcessMaterial（カスタム後処理マテリアル）

### 役割

ユーザーが UE マテリアルエディタで作成した**ポストプロセスマテリアル**を実行する。  
`Blendable` インターフェース経由でパイプラインの任意の位置に挿入できる。

```cpp
struct FPostProcessMaterialParameters
{
    // 後処理マテリアルが参照できるシーンテクスチャ群
    FRDGTextureRef SceneColor;
    FRDGTextureRef SceneDepth;
    FRDGTextureRef SceneNormal;
    FRDGTextureRef SceneVelocity;
    // + GBuffer A/B/C/D, CustomDepth 等
};

// 単一マテリアルをパス追加
FScreenPassTexture AddPostProcessMaterialPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FPostProcessMaterialParameters& Parameters,
    const UMaterialInterface* Material,
    FScreenPassTexture Input);

// マテリアルチェーンを一括追加（ブレンダブルロケーション別）
FScreenPassTexture AddPostProcessMaterialChain(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FPostProcessMaterialParameters& Parameters,
    FScreenPassTexture Input,
    EBlendableLocation Location,
    const TArray<UMaterialInterface*>& Materials);
```

---

## 2. PostProcessSubsurface（サブサーフェス散乱後処理）

### 役割

GBuffer の **Subsurface（SSS）情報**を後処理パスでブラーして皮膚・ろうそく等の透光感を表現する。  
Substrate 有効時は Substrate 側で処理されるため、このパスは旧シェーディングモデル向け。

```cpp
// SSS が有効か
bool IsSubsurfaceEnabled(const FViewInfo& View);

// SSS 後処理パスを追加
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

---

## 3. PostProcessLensFlares（レンズフレア）

### 役割

カメラレンズの**フレア・ゴースト・Dirt Mask**を生成する。

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
    FScreenPassTexture Bloom;          // Bloom テクスチャ（フレアのソース）
    FRDGTextureRef BokehShapeTexture;  // ボケ形状テクスチャ（オプション）
    FLinearColor TintColors[8];        // フレア各ゴーストの色
    float Intensity;
    float Threshold;
    float BokehSize;
};

FScreenPassTexture AddLensFlaresPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Bloom,
    const FLensFlareInputs& Inputs);
```

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.LensFlareQuality` | 2 | レンズフレア品質 |

---

## 4. PostProcessUpscale（空間的アップスケール）

### 役割

主に**TSR/TAA 後の最終アップスケール**や、シンプルな解像度変換に使用する。  
外部プラグイン（DLSS/XeSS 等）との差し替えインターフェースも提供。

```cpp
enum class EUpscaleMethod : uint8
{
    None          = 0,
    Nearest       = 1,
    Bilinear      = 2,
    Bicubic       = 3,
    BicubicSharper = 4,
    CatmullRom    = 5,
    Lanczos       = 6,
    Gaussian      = 7,
    SmoothStep    = 8,
    Temporal      = 9,
    TSR           = 10,
};

enum class EUpscaleStage : uint8
{
    PrimaryToSecondary,   // 一次 → 二次解像度
    PrimaryToOutput,      // 一次 → 最終出力解像度
    SecondaryToOutput,    // 二次 → 最終出力解像度
};

// 空間アップスケーラーのインターフェース（プラグイン差し替え用）
class ISpatialUpscaler
{
public:
    virtual FScreenPassTexture AddPasses(
        FRDGBuilder& GraphBuilder,
        const FViewInfo& View,
        const FInputs& Inputs) const = 0;

    virtual const TCHAR* GetDebugName() const = 0;
    virtual ISpatialUpscaler* Fork_GameThread(const FSceneViewFamily&) const = 0;
};
```

---

## 5. NeuralPostProcess（ニューラルネット後処理）

### 役割

**NNE（Neural Network Engine）**を使用したニューラルネット後処理（実験的機能）。  
ニューラルモデルを読み込み、GPU 上で推論を実行する。

```cpp
struct FNeuralPostProcessResource
{
    // ニューラルネット推論リソース
    TSharedPtr<UE::NNE::IModelInstanceGPU> ModelInstance;
    TArray<FRDGTextureRef> IntermediateTextures;
};

// 必要なリソースを確保（初回実行時）
void AllocateNeuralPostProcessingResourcesIfNeeded(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FNeuralPostProcessResource& OutResource);

// ニューラル後処理を適用
FScreenPassTexture ApplyNeuralPostProcess(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input,
    FNeuralPostProcessResource& Resource);
```

---

## 6. PostProcessAA / SMAA / MitchellNetravali

| ファイル | 役割 |
|---------|------|
| `PostProcessAA.cpp` | FXAA（高速近似AA）のパス追加 |
| `SubpixelMorphologicalAA.cpp` | SMAA（形態学的AA、高品質エッジ検出）|
| `PostProcessMitchellNetravali.h/.cpp` | Mitchell-Netravali フィルター（アップスケール用高品質リサンプル）|

```cpp
// FXAA パス
FScreenPassTexture AddPostProcessAA(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input);
```

---

## 7. その他ユーティリティ

| ファイル | 役割 |
|---------|------|
| `AlphaInvert.h/.cpp` | アルファチャンネル反転（合成用） |
| `LensDistortion.cpp` | レンズディストーション補正 |
| `SceneFilterRendering.h/.cpp` | フルスクリーンパス描画の共通ユーティリティ（DrawRectangle等） |
| `DebugAlphaChannel.h/.cpp` | アルファチャンネルデバッグ表示 |

---

## 主要 CVar まとめ

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.SSS.Quality` | 1 | サブサーフェース品質（0=Disabled） |
| `r.SSS.Filter` | 1 | SSS フィルタータイプ |
| `r.SMAA.Quality` | 1 | SMAA 品質 |
| `r.Upscale.Quality` | 3 | 空間アップスケール品質 |
| `r.NeuralPostProcess.Enable` | 0 | ニューラル後処理有効 |

---

## コード実行フロー

### PostProcessMaterial エントリポイント

```
AddPostProcessingPasses()
  └─ AddPostProcessMaterialChain()
       └─ IBlendableInterface::OverrideBlendableSettings() から材料リストを収集
            ├─ BL_BeforeBloom ロケーション → Bloom 前に PS/CS で実行
            ├─ BL_BeforeTonemap ロケーション → Tonemap 前（HDR 空間）
            └─ BL_AfterTonemap ロケーション → Tonemap 後（LDR 空間）
```

### PostProcessSubsurface エントリポイント

```
FDeferredShadingSceneRenderer::Render()
  └─ RenderSubsurface()
       ├─ SubsurfaceCombinePass CS: Diffuse + Specular を分離
       ├─ SubsurfaceFilterPass CS: 複数パスの Gaussian フィルター
       └─ SubsurfaceRecombinePass: 最終合成
```

### フロー詳細

1. **PostProcessMaterial** — `UMaterialInterface` が `IBlendableInterface` を実装し、`EBlendableLocation` で挿入位置を指定する。`AddPostProcessMaterialChain()` が該当位置のマテリアルを順番に実行する
2. **Subsurface** — ベースパス後（RenderSubsurface）に実行される。DiffusionProfile に従ってカーネルを変え皮膚・蝋等の散乱を表現する
3. **NeuralPostProcess** — NNE（Neural Network Engine）に GPU 推論を委譲する。入力/出力バッファは RDG テクスチャとして扱われる
4. **Upscale** — `ISpatialUpscaler` インターフェース経由で空間アップスケール（Lanczos / Catmull-Rom 等）を実行する

### 関与クラス・関数一覧

| クラス/関数 | 役割 |
|------------|------|
| `AddPostProcessMaterialChain()` | カスタム PP マテリアルの連鎖実行 |
| `RenderSubsurface()` | サブサーフェース散乱後処理 |
| `ISpatialUpscaler::AddPasses()` | 空間アップスケール |
| `IBlendableInterface` | PP マテリアル挿入位置の宣言 |

---

## 関連リファレンス

- [[ref_pp_material_misc]] — PostProcessMaterial / Subsurface / LensFlares / Upscale リファレンス
- [[ref_pp_orchestrator]] — ブレンダブルロケーションとパイプライン���体
