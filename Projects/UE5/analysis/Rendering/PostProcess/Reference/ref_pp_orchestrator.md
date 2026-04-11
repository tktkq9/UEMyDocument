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

---

> [!note]- AddPostProcessingPasses の FScreenPassTexture チェーン
> 各パスは `FScreenPassTexture` を入出力として連鎖する。前パスの出力が次パスの入力になるため、  
> `AddPostProcessingPasses()` (`PostProcessing.cpp:347`) 内でローカル変数 `SceneColor` が各パスの呼び出しのたびに更新される。  
> パスが無効な場合（品質設定 = 0 等）は入力をそのまま返すため、呼び出し側は常に最新の `FScreenPassTexture` を次パスに渡せばよい。

> [!note]- EBlendableLocation と挿入タイミングの注意
> `BL_BeforeTonemap`（HDR 空間）と `BL_AfterTonemap`（LDR 空間）は大きく異なる。  
> `BeforeTonemap` ではシーンの HDR 値（0〜数千 nit）を直接操作できるが、  
> `AfterTonemap` では 0〜1 に収まった LDR 値のみを扱う。  
> ユーザーが HDR 操作をしたい場合（輝度調整等）は `BeforeTonemap` を使わなければならない。

> [!note]- r.PostProcessing.PreferCompute の効果
> `r.PostProcessing.PreferCompute = 1` を設定すると、サポートされるパスが Vertex/Pixel Shader から  
> Compute Shader 版に切り替わる。コンソールや GPU によっては CS のほうが効率的な場合があるが、  
> すべてのパスに CS 版があるわけではない。未対応パスは引き続き VS/PS を使う。
