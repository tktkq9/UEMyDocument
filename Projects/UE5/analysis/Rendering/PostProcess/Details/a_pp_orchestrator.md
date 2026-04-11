# PostProcess a: オーケストレーター

- 対象: `PostProcess/PostProcessing.h/.cpp`
- 上位: [[05_postprocess_overview]]

---

## 役割

`AddPostProcessingPasses()` が**全ポストプロセスパスを RDG に順番に登録する**オーケストレーター関数。  
`FDeferredShadingRenderer::Render()` から呼ばれ、SceneColor が確定した後に実行される。

---

## 入力データ構造

```cpp
struct FPostProcessingInputs
{
    FRDGTextureRef SceneColor;           // HDR SceneColor（ライティング結果）
    FRDGTextureRef SceneDepth;           // シーン深度
    FRDGTextureRef SceneVelocity;        // ベロシティバッファ（TAA/MotionBlur用）
    FRDGTextureRef SeparateTranslucency; // 分離半透明
    TRDGUniformBufferRef<FSceneTextureUniformParameters> SceneTextures;
};
```

---

## 全パスの実行順序

```
AddPostProcessingPasses()
│
├─ [A] MotionBlur
│     → AddMotionBlurPass()
│
├─ [B] Temporal AA / TSR
│     → AddTemporalAAPass()    : TAA（r.AntiAliasingMethod=2）
│     → ITemporalUpscaler      : TSR/DLSS等（r.AntiAliasingMethod=4）
│
├─ [C] Separate Translucency Composition
│
├─ [D] Post Process Materials (Before Bloom)
│     → AddPostProcessMaterialChain()  ブレンダブルロケーション=BL_BeforeBloom
│
├─ [E] Bloom
│     → AddBloomSetupPass() + AddGaussianBloomPasses()
│     → AddFFTBloomPass()  ← r.BloomMethod=1 の場合
│
├─ [F] Lens Flares
│     → AddLensFlaresPass()
│
├─ [G] Exposure / Eye Adaptation
│     → AddHistogramPass()
│     → AddHistogramEyeAdaptationPass() or AddBasicEyeAdaptationPass()
│
├─ [H] Depth of Field
│     → DiaphragmDOF::AddPasses()
│
├─ [I] Local Exposure
│     → AddLocalExposureBlurredLogLuminancePass()
│
├─ [J] Post Process Materials (Before Tonemap)
│     → AddPostProcessMaterialChain()  BL_BeforeTonemap
│
├─ [K] Color Grading / Tonemap
│     → AddCombineLUTPass()
│     → AddTonemapPass()
│
├─ [L] Post Process Materials (After Tonemap)
│     → AddPostProcessMaterialChain()  BL_AfterTonemap
│
├─ [M] FXAA / SMAA（シンプルAA）
│     → AddPostProcessAA()
│
├─ [N] Upscale
│     → AddSpatialUpscalePass() or ISpatialUpscaler::AddPasses()
│
└─ [O] Debug / Editor Compositing
      → AddDebugViewPostProcessingPasses()
      → SelectionOutline / GBufferHints / VisualizeBuffer 等
```

---

## 主要関数シグネチャ

```cpp
// メインエントリポイント
void AddPostProcessingPasses(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    int32 ViewIndex,
    FSceneUniformBuffer& SceneUniformBuffer,
    EDiffuseIndirectMethod DiffuseIndirectMethod,
    EReflectionsMethod ReflectionsMethod,
    const FPostProcessingInputs& Inputs,
    const Nanite::FRasterResults* NaniteRasterResults,
    FVirtualShadowMapArray* VirtualShadowMapArray,
    ...);

// デバッグビュー専用パス（VisualizeBuffer 等）
void AddDebugViewPostProcessingPasses(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FPostProcessingInputs& Inputs,
    const Nanite::FRasterResults* NaniteRasterResults);

// モバイル向けポストプロセス
void AddMobilePostProcessingPasses(
    FRDGBuilder& GraphBuilder,
    FScene* Scene,
    const FViewInfo& View,
    int32 ViewIndex,
    const FMobilePostProcessingInputs& Inputs,
    FInstanceCullingManager& InstanceCullingManager);

// 有効化チェック
bool IsPostProcessingEnabled(const FViewInfo& View);
```

---

## Post Process Material のブレンダブルロケーション

```
BL_BeforeTranslucency   … 半透明描画前
BL_BeforeDOF            … DOF前
BL_BeforeBloom          … Bloom前
BL_BeforeTonemap        … Tonemap前（HDRで動作）
BL_AfterTonemap         … Tonemap後（LDRで動作）
BL_ReplacingTonemapper  … Tonemapperを置き換え
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.PostProcessing.PropagateAlpha` | 0 | アルファ伝搬モード |
| `r.PostProcessing.PreferCompute` | 0 | Compute シェーダー優先 |
| `r.PostProcessing.ForceAsyncDispatch` | 0 | 非同期ディスパッチ強制 |
| `r.AntiAliasingMethod` | 4 | AA方式（0=None,1=FXAA,2=TAA,4=TSR） |

---

## コード実行フロー

### エントリポイント

```
FDeferredShadingRenderer::Render()
  └─ AddPostProcessingPasses()           PostProcessing.cpp:347
       └─ （15パスを順番に RDG に登録）
            ├─ MotionBlur               PostProcessMotionBlur.cpp:1333
            ├─ TAA / TSR               TemporalAA.cpp:571
            ├─ PostProcessMaterial(BeforeBloom)
            ├─ Bloom                   PostProcessBloomSetup.cpp:120
            ├─ LensFlares
            ├─ Histogram → EyeAdaptation
            ├─ DiaphragmDOF             DiaphragmDOF.cpp:1486
            ├─ LocalExposure
            ├─ PostProcessMaterial(BeforeTonemap)
            ├─ CombineLUT → Tonemap    PostProcessCombineLUTs.cpp:494 / PostProcessTonemap.cpp:569
            ├─ PostProcessMaterial(AfterTonemap)
            ├─ FXAA / SMAA
            ├─ Upscale
            └─ AddDebugViewPostProcessingPasses()  PostProcessing.cpp:2073
```

### フロー詳細

1. `AddPostProcessingPasses()` (`PostProcessing.cpp:347`) が呼ばれ、`FPostProcessingInputs` から SceneColor 等を受け取る
2. 各パスは `FScreenPassTexture` を入出力として連鎖し、前パスの出力が次パスの入力になる
3. `r.PostProcessing.PreferCompute = 1` の場合、Compute シェーダー版パスが選択される
4. デバッグビューモードでは `AddDebugViewPostProcessingPasses()` が通常パイプラインの代わりに実行される
5. モバイルでは `AddMobilePostProcessingPasses()` が呼ばれ、軽量化された別パイプラインが実行される

### 関与クラス・関数一覧

| クラス/関数 | ファイル:行 | 役割 |
|------------|-----------|------|
| `AddPostProcessingPasses()` | `PostProcessing.cpp:347` | 全パスオーケストレーター |
| `AddDebugViewPostProcessingPasses()` | `PostProcessing.cpp:2073` | デバッグビュー専用パス |
| `AddMobilePostProcessingPasses()` | `PostProcessing.cpp:2440` | モバイル向け軽量パイプライン |
| `IsPostProcessingEnabled()` | `PostProcessing.h` | PP 有効/無効判定 |
| `FPostProcessingInputs` | `PostProcessing.h` | パイプライン入力データ |
| `FScreenPassTexture` | `ScreenPass.h` | パス間のテクスチャ受け渡し |

---

## 関連リファレンス

- [[ref_pp_orchestrator]] — AddPostProcessingPasses / FPostProcessingInputs リファレンス
- [[ref_pp_taa]] — TAA リファレンス
- [[ref_pp_tsr]] — TSR リファレンス
