# ref: MegaLights コアクラス群

- 対象: `MegaLights.h` / `MegaLightsInternal.h` / `MegaLights.cpp`
- Details: [[a_megalights_pipeline]] | [[c_megalights_shadow]]

---

## EMegaLightsMode（動作モード列挙）

```cpp
// MegaLights.h（概略）
enum class EMegaLightsMode : uint8
{
    Disabled,    // MegaLights 無効（or ライスト非対応）
    EnabledRT,   // HW Ray Tracing によるシャドウ
    EnabledVSM,  // Virtual Shadow Maps によるシャドウ
};

// モード選択ロジック（MegaLights.cpp）
EMegaLightsMode MegaLights::GetMegaLightsMode(
    const FSceneViewFamily& ViewFamily,
    uint8 LightType,
    bool bLightAllowsMegaLights,
    TEnumAsByte<EMegaLightsShadowMethod::Type> ShadowMethod)
{
    if (!IsEnabled(ViewFamily) || !bLightAllowsMegaLights)
        return EMegaLightsMode::Disabled;
    if (UseHardwareRayTracing(ViewFamily))
        return EMegaLightsMode::EnabledRT;
    return EMegaLightsMode::EnabledVSM;
}
```

---

## FMegaLightsVolume（ライティングボリューム）

```cpp
// MegaLights.h（概略）
class FMegaLightsVolume
{
    FRDGTextureRef Texture;                           // 不透明ライティング結果
    FRDGTextureRef TranslucencyAmbient[TVC_MAX];      // 半透明アンビエント（カスケード別）
    FRDGTextureRef TranslucencyDirectional[TVC_MAX];  // 半透明ディレクショナル（カスケード別）
    // TVC_MAX = TranslucencyVolumeCascadeCount（デフォルト 2）
};
```

---

## FMegaLightsFrameTemporaries（フレーム単位一時データ）

```cpp
// MegaLights.cpp:1833
struct FMegaLightsFrameTemporaries
{
    // ビューごとのコンテキスト（通常 GBuffer 用）
    TArray<FMegaLightsViewContext, SceneRenderingAllocator> ViewContexts;
    // Hair Strands 専用コンテキスト
    TArray<FMegaLightsViewContext, SceneRenderingAllocator> ViewContextsHairStrands;
};
// GenerateMegaLightsSamples() で生成、RenderMegaLights() で消費
// TSharedPtr で寿命管理
```

---

## FMegaLightsViewContext（ビュー単位コンテキスト）

```cpp
// MegaLightsInternal.h:198
class FMegaLightsViewContext
{
public:
    FMegaLightsViewContext(
        FRDGBuilder& InGraphBuilder,
        const int32 InViewIndex,
        const FViewInfo& InView,
        const FSceneViewFamily& InViewFamily,
        const FScene* InScene,
        const FSceneTextures& InSceneTextures,
        bool bInUseVSM);

    // タイル分類マークテクスチャ取得（ShadingPassIndex 別）
    FRDGTextureRef TileClassificationMark(uint32 ShadingPassIndex);

    // ---- フェーズメソッド ----
    void Setup(FRDGTextureRef LightingChannelsTexture, ...);
    void GenerateSamples(FRDGTextureRef LightingChannelsTexture, uint32 ShadingPassIndex);
    void MarkVSMPages(const FVirtualShadowMapArray& VirtualShadowMapArray);
    void RayTrace(const FVirtualShadowMapArray&, const TArrayView<FRDGTextureRef>&, uint32 ShadingPassIndex);
    void Resolve(FRDGTextureRef OutputColorTarget, FMegaLightsVolume* MegaLightsVolume, uint32 ShadingPassIndex);
    void DenoiseLighting(FRDGTextureRef OutputColorTarget);

    bool AreSamplesGenerated() const;
};
```

---

## MegaLights::ETileType（タイル種別）

```cpp
// MegaLightsInternal.h:143
enum class ETileType : uint8
{
    // レガシー GBuffer 用
    SimpleShading,          // シンプルなシェーディングモデル
    ComplexShading,         // 複雑なシェーディングモデル
    SimpleShading_Rect,     // Rect Light + Simple
    ComplexShading_Rect,    // Rect Light + Complex
    SimpleShading_Rect_Textured,
    ComplexShading_Rect_Textured,

    // Substrate 用
    SingleShading,          // Substrate Single
    ComplexSpecialShading,  // Substrate Complex/Special
    SingleShading_Rect,
    ComplexSpecialShading_Rect,
    SingleShading_Rect_Textured,
    ComplexSpecialShading_Rect_Textured,

    Empty,  // 処理不要タイル
};
```

---

## 名前空間別公開 API

| 名前空間 | 主要関数 | 役割 |
|---------|---------|------|
| `MegaLights::` | `IsEnabled()` / `GetMegaLightsMode()` | 有効判定・モード選択 |
| `MegaLights::` | `UseHardwareRayTracing()` | HW RT 使用判定 |
| `MegaLights::` | `GetNumSamplesPerPixel2d()` | サンプル数計算 |
| `MegaLights::` | `UseSpatialFilter()` / `UseTemporalFilter()` | フィルタ有効判定 |
| `MegaLightsVolume::` | `UsesLightFunction()` | ボリューム LF 使用判定 |
| `MegaLightsTranslucencyVolume::` | `UsesLightFunction()` | 半透明ボリューム LF 使用判定 |

---

> [!note]- FMegaLightsViewContext のライフサイクル
> 1. `GenerateMegaLightsSamples()` でサンプル生成 → `FMegaLightsFrameTemporaries` に格納
> 2. `RenderMegaLights()` で shadow/resolve/denoise を呼び出す
> 3. フレーム終了時に `TSharedPtr` がスコープを抜けて自動解放
> 2フレームにわたる寿命はなく、`FMegaLightsViewState` がHistoryを保持する。

> [!note]- bUseVSM の判定
> `FMegaLightsViewContext` コンストラクタに渡される `bInUseVSM` は
> `ShadowSceneRenderer.AreAnyLightsUsingMegaLightsVSM()` の戻り値。
> true の場合、シャドウパスで `MarkVSMPages()` が呼ばれ、
> 既存の VSM Shadow Map テクスチャからシャドウ値を参照する。
