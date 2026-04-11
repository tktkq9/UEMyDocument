# リファレンス：IndirectLightRendering.cpp（最終ライティング合成）

- グループ: g - Final Composite
- 上位: [[g_lumen_final_composite]]
- 関連: [[ref_lumen_diffuse_indirect]] | [[ref_lumen_reflections]]
- ソース: `Engine/Source/Runtime/Renderer/Private/IndirectLightRendering.cpp`, `DeferredShadingRenderer.h`

---

## 概要

Lumen（および SSGI / Plugin）の最終ライティング合成を担うファイル。  
Diffuse GI・Reflections・AO テクスチャを受け取り、GBuffer と組み合わせて SceneColor に加算書き込みする。

---

## EDiffuseIndirectMethod

```cpp
// DeferredShadingRenderer.h:290
enum class EDiffuseIndirectMethod
{
    Disabled,  // 間接照明なし
    Lumen,     // Lumen GI（Screen Probe Gather）
    SSGI,      // Screen Space GI
    Plugin,    // IScreenSpaceDenoiser プラグイン
};
```

---

## ELumenIndirectLightingSteps

```cpp
// DeferredShadingRenderer.h:115
enum class ELumenIndirectLightingSteps
{
    None              = 0,
    ScreenProbeGather = 1u << 0,  // Diffuse GI 計算ステップ
    Reflections       = 1u << 1,  // 反射計算ステップ
    Composite         = 1u << 3,  // 最終合成ステップ
    All               = ScreenProbeGather | Reflections | Composite
};
ENUM_CLASS_FLAGS(ELumenIndirectLightingSteps)
```

フレームごとに `StepsLeft` フラグを減らしていき、最終的に `Composite` のみで合成を実行する。

---

## FAsyncLumenIndirectLightingOutputs

```cpp
// DeferredShadingRenderer.h:125
struct FAsyncLumenIndirectLightingOutputs
{
    struct FViewOutputs
    {
        FSSDSignalTextures                                    IndirectLightingTextures;      // GI + 反射テクスチャ群
        FLumenMeshSDFGridParameters                           MeshSDFGridParameters;         // SDF グリッドパラメータ
        LumenRadianceCache::FRadianceCacheInterpolationParameters RadianceCacheParameters;   // Radiance Cache 補間パラメータ
        FLumenScreenSpaceBentNormalParameters                 ScreenBentNormalParameters;    // Bent Normal AO パラメータ
    };

    TArray<FViewOutputs, TInlineAllocator<1>> ViewOutputs;                  // ビューごとの出力
    ELumenIndirectLightingSteps               StepsLeft = ELumenIndirectLightingSteps::All; // 残りステップ
    bool                                      bHasDrawnBeforeLightingDecals = false;

    // AsyncCompute 完了時に呼ぶ（完了したステップを除去）
    void DoneAsync(bool bAsyncReflections);

    // Direct Lighting パス完了時に呼ぶ
    void DonePreLights();

    // Composite 完了時に呼ぶ
    void DoneComposite();
};
```

---

## FDiffuseIndirectCompositePS

```cpp
// IndirectLightRendering.cpp:119
class FDiffuseIndirectCompositePS : public FGlobalShader
{
public:
    DECLARE_GLOBAL_SHADER(FDiffuseIndirectCompositePS)
    SHADER_USE_PARAMETER_STRUCT(FDiffuseIndirectCompositePS, FGlobalShader)

    // ---- パーミュテーション ----
    class FApplyDiffuseIndirectDim : SHADER_PERMUTATION_INT("DIM_APPLY_DIFFUSE_INDIRECT", 3);
    // 0 = 合成なし（パス構造のみ）
    // 1 = SSGI / Plugin
    // 2 = Lumen（Screen Probe Gather）

    class FUpscaleDiffuseIndirectDim : SHADER_PERMUTATION_BOOL("DIM_UPSCALE_DIFFUSE_INDIRECT");
    // SSGI のみ有効（Lumen では false）

    class FScreenBentNormalMode : SHADER_PERMUTATION_RANGE_INT("DIM_SCREEN_BENT_NORMAL_MODE", 0, 3);
    // Lumen の Screen Bent Normal AO 適用モード

    class FShortRangeGI : SHADER_PERMUTATION_BOOL("DIM_SHORT_RANGE_GI");
    // Lumen 近距離 GI の有無（ShortRangeAO 兼用）

    class FEnableDualSrcBlending : SHADER_PERMUTATION_BOOL("ENABLE_DUAL_SRC_BLENDING");
    // Dual Source Blending 対応 GPU の場合 SceneColor コピーを省略

    class FScreenSpaceReflectionTiledComposition : SHADER_PERMUTATION_BOOL("SSR_TILED_COMPOSITION");
    // SSR タイル合成パス（Lumen + SSR 混在ケース）

    // ---- 主要入力パラメータ ----
    SHADER_PARAMETER_RDG_TEXTURE(Texture2DArray, DiffuseIndirect_Lumen_0)  // Diffuse GI メイン
    SHADER_PARAMETER_RDG_TEXTURE(Texture2DArray, DiffuseIndirect_Lumen_1)  // Backface Diffuse
    SHADER_PARAMETER_RDG_TEXTURE(Texture2DArray, DiffuseIndirect_Lumen_2)  // Short Range GI
    SHADER_PARAMETER_RDG_TEXTURE(Texture2DArray, DiffuseIndirect_Lumen_3)  // Reflections

    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, AmbientOcclusionTexture)        // SSAO / RTAO マスク
    SHADER_PARAMETER(float, AmbientOcclusionStaticFraction)                  // 静的 AO のブレンド率

    SHADER_PARAMETER_TEXTURE(Texture2D, PreIntegratedGF)                     // BRDF 事前積分テーブル
    SHADER_PARAMETER(float, LumenReflectionSpecularScale)                    // 反射強度スケール
    SHADER_PARAMETER(float, LumenReflectionContrast)                         // 反射コントラスト

    // Substrate 対応（不透明粗面の屈折 + サブサーフェス分離出力）
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float3>, OutOpaqueRoughRefractionSceneColor)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float3>, OutSubSurfaceSceneColor)

    RENDER_TARGET_BINDING_SLOTS()  // SceneColor RT
};
// シェーダーファイル: /Engine/Private/DiffuseIndirectComposite.usf
// エントリポイント: MainPS
```

---

## FAmbientCubemapCompositePS

```cpp
// IndirectLightRendering.cpp:231
class FAmbientCubemapCompositePS : public FGlobalShader
{
    // アンビエントキューブマップ（PostProcessVolume 設定）を SceneColor に加算
    // ReflectionCapture とは別：ポストプロセス的なエリアライト補完
};
// シェーダーファイル: /Engine/Private/AmbientCubemapComposite.usf
```

---

## 主要関数

### CommitIndirectLightingState

```cpp
// IndirectLightRendering.cpp:484
void FDeferredShadingSceneRenderer::CommitIndirectLightingState();
```

フレーム開始時に各ビューの `FPerViewPipelineState` を確定する。

| 条件 | 設定値 |
|-----|--------|
| `ShouldRenderLumenDiffuseGI()` | `DiffuseIndirectMethod = Lumen` |
| `IsScreenSpaceDiffuseIndirectSupported()` | `DiffuseIndirectMethod = SSGI` |
| `ShouldRenderPluginGlobalIllumination()` | `DiffuseIndirectMethod = Plugin` |

---

### DispatchAsyncLumenIndirectLightingWork

```cpp
// IndirectLightRendering.cpp:889
void FDeferredShadingSceneRenderer::DispatchAsyncLumenIndirectLightingWork(
    FRDGBuilder& GraphBuilder,
    FSceneTextures& SceneTextures,
    FInstanceCullingManager& InstanceCullingManager,
    FLumenSceneFrameTemporaries& LumenFrameTemporaries,
    FDynamicShadowsTaskData* DynamicShadowsTaskData,
    FRDGTextureRef LightingChannelsTexture,
    FAsyncLumenIndirectLightingOutputs& Outputs);
```

- `LumenDiffuseIndirect::UseAsyncCompute()` が true の場合のみ実行
- `ERDGPassFlags::AsyncCompute` で GI（ScreenProbeGather）と反射を並列発行
- 完了後 `Outputs.DoneAsync(bAsyncComputeReflections)` でフラグ更新

---

### RenderDiffuseIndirectAndAmbientOcclusion

```cpp
// IndirectLightRendering.cpp:977
void FDeferredShadingSceneRenderer::RenderDiffuseIndirectAndAmbientOcclusion(
    FRDGBuilder& GraphBuilder,
    FSceneTextures& SceneTextures,
    FLumenSceneFrameTemporaries& LumenFrameTemporaries,
    FRDGTextureRef LightingChannelsTexture,
    bool bCompositeRegularLumenOnly,  // 合成ステップのみ実行するか
    bool bIsVisualizePass,            // デバッグビジュアライズか
    FAsyncLumenIndirectLightingOutputs& AsyncLumenIndirectLightingOutputs);
```

- `bCompositeRegularLumenOnly=true` の場合は AsyncCompute 完了後の合成のみ実行
- AO（SSAO / RTAO）も同じパス内で処理
- `IScreenSpaceDenoiser::DenoiseReflections()` は Lumen では `EMode::Disabled` のため呼ばれない

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Lumen.DiffuseIndirect.Allow` | 1 | Lumen GI を有効にするか |
| `r.Lumen.Reflections.Allow` | 1 | Lumen 反射を有効にするか |
| `r.Lumen.AsyncCompute` | 1 | AsyncCompute キューで並列実行するか |
| `r.Lumen.DiffuseIndirect.Visualize` | 0 | GI 合成前テクスチャのデバッグ表示 |
| `r.DiffuseIndirectForceCopyPass` | 0 | Dual Source Blending を無効化し SceneColor コピーを強制 |

---

> [!note]- AsyncCompute と StepsLeft フラグの関係
> `DispatchAsyncLumenIndirectLightingWork()` は GBuffer 描画の前に呼ばれ、Lumen GI と反射を AsyncCompute キューで Graphics と並列実行する。  
> 完了すると `FAsyncLumenIndirectLightingOutputs::DoneAsync()` で `StepsLeft` から `ScreenProbeGather`（と反射）フラグが除去される。  
> その後 Direct Lighting パス完了時に `DonePreLights()` が呼ばれ、最後に `RenderDiffuseIndirectAndAmbientOcclusion()` 内で `Composite` ステップのみが残った状態で合成が実行される。

> [!note]- FDiffuseIndirectCompositePS のパーミュテーション爆発対策
> `FApplyDiffuseIndirectDim=2`（Lumen）の場合のみ `FScreenBentNormalMode` や `FShortRangeGI` が意味を持つ。  
> `ShouldCompilePermutation()` でこれらの組み合わせを弾き、Lumen 以外で無効なパーミュテーションをコンパイルしないようにしている（`DIM_APPLY_DIFFUSE_INDIRECT != 2` のとき `FScreenBentNormalMode != 0 || FShortRangeGI` は false を返す）。  
> Substrate 対応のタイルパーミュテーションも同様で、`DIM_APPLY_DIFFUSE_INDIRECT == 2` の場合のみコンパイルされる。

> [!note]- Dual Source Blending と SceneColor コピーパス
> `FEnableDualSrcBlending=true` のハードウェア（PC 向け GPU の多く）では、`FDiffuseIndirectCompositePS` が SceneColor の現在値を **ブレンドステート** で直接読み書きする（1パス）。  
> 非対応の場合（モバイル等）は `AddCopyTexturePass()` で SceneColor のコピーを作成し `PSParameters->SceneColorTexture` にバインドしてから描画する（2パス）。  
> `r.DiffuseIndirectForceCopyPass=1` でコピーパスを強制できる（デバッグ用）。
