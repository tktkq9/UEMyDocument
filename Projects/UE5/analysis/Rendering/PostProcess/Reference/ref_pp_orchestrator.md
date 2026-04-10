# REF: PostProcessing.h / .cpp

- 対象ファイル: `Private/PostProcess/PostProcessing.h` / `.cpp`
- 関連Details: [[a_pp_orchestrator]]

---

## 構造体

### `FPostProcessingInputs`

PostProcess パイプライン全体への入力データ。

```cpp
struct FPostProcessingInputs
{
    FRDGTextureRef SceneColor;           // HDR SceneColor（ライティング後）
    FRDGTextureRef SceneDepth;           // シーン深度
    FRDGTextureRef SceneVelocity;        // ベロシティバッファ（TAA/MotionBlur用）
    FRDGTextureRef SeparateTranslucency; // 分離半透明テクスチャ
    FRDGTextureRef SeparateModulation;   // 分離モジュレーション
    TRDGUniformBufferRef<FSceneTextureUniformParameters> SceneTextures;

    // 検証
    void Validate() const;
};
```

### `FMobilePostProcessingInputs`

モバイル向けの PostProcess 入力データ。

```cpp
struct FMobilePostProcessingInputs
{
    FRDGTextureRef SceneColor;
    FRDGTextureRef SceneDepth;
    FRDGTextureRef SeparateTranslucency;
    bool bViewFamilyOutputInHDR;
};
```

---

## 主要関数

### メインポストプロセス

```cpp
// 全ポストプロセスパスを RDG に登録するオーケストレーター
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
    FInstanceCullingManager& InstanceCullingManager,
    FRDGTextureRef HairDensityTexture = nullptr,
    FRDGTextureRef HairDepthTexture = nullptr);
```

### デバッグビュー

```cpp
// VisualizeBuffer / GBufferHints 等のデバッグ専用パス
void AddDebugViewPostProcessingPasses(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FPostProcessingInputs& Inputs,
    const Nanite::FRasterResults* NaniteRasterResults);
```

### モバイル

```cpp
// モバイル向けポストプロセス（軽量化パイプライン）
void AddMobilePostProcessingPasses(
    FRDGBuilder& GraphBuilder,
    FScene* Scene,
    const FViewInfo& View,
    int32 ViewIndex,
    const FMobilePostProcessingInputs& Inputs,
    FInstanceCullingManager& InstanceCullingManager);
```

### ユーティリティ

```cpp
// ポストプロセスが有効か（r.PostProcessing.Disable や r.PostProcessAAQuality等を考慮）
bool IsPostProcessingEnabled(const FViewInfo& View);

// ワイヤーフレームやデバッグ表示でPPを無効化すべきか
bool IsPostProcessingWithoutAA(const FViewInfo& View);
```

---

## ブレンダブルロケーション

```cpp
// Post Process Material の挿入位置
enum class EBlendableLocation : uint8
{
    BL_AfterTonemapping        = 0, // Tonemap 後（LDR）
    BL_BeforeTonemapping       = 1, // Tonemap 前（HDR）
    BL_BeforeTranslucency      = 2, // 半透明前
    BL_AfterMotionBlur         = 3, // MotionBlur 後
    BL_TranslucencyAfterDOF    = 4, // DOF後半透明
    BL_SSRInput                = 5, // SSR 入力
    BL_ReplacingTonemapper     = 6, // Tonemap 置き換え
    BL_BeforeTranslucencyAndDOF = 7,
};
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.PostProcessing.PropagateAlpha` | 0 | アルファ伝搬（0=無効, 1=線形, 2=プリマル）|
| `r.PostProcessing.PreferCompute` | 0 | Compute シェーダー優先実行 |
| `r.PostProcessing.ForceAsyncDispatch` | 0 | 非同期ディスパッチ強制 |
| `r.PostProcessing.Disable` | 0 | 全ポストプロセス無効（デバッグ） |
| `r.PostProcessAAQuality` | 4 | AA 品質（ポストプロセス）|
| `r.AntiAliasingMethod` | 4 | AA 方式（0=None,1=FXAA,2=TAA,4=TSR） |
