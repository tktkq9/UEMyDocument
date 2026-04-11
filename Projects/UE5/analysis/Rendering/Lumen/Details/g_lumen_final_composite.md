# Lumen 最終ライティング合成（Final Lighting Composite）

- 上位: [[02_lumen_overview]]
- 関連: [[e_lumen_diffuse_gi]] | [[f_lumen_reflections]]

---

## 概要

Lumen の最終ステップ。  
Diffuse GI・Reflections・AO の各テクスチャを GBuffer と合成して SceneColor に書き込む。  
実行方法は `EDiffuseIndirectMethod` によって分岐し、Lumen / SSGI / Plugin の3方式をサポートする。

**処理の主役は2つ：**

| フェーズ | 関数 | ファイル |
|---------|------|---------|
| GPU 計算（非同期） | `DispatchAsyncLumenIndirectLightingWork()` | `IndirectLightRendering.cpp:889` |
| 合成（GBuffer への書き込み） | `RenderDiffuseIndirectAndAmbientOcclusion()` | `IndirectLightRendering.cpp:977` |

---

## 処理方式の選択（CommitIndirectLightingState）

フレーム開始時に `CommitIndirectLightingState()`（`IndirectLightRendering.cpp:484`）が  
各ビューの `FPerViewPipelineState` に処理方式を決定する。

```
CommitIndirectLightingState()
  ├─ ShouldRenderLumenDiffuseGI()          → EDiffuseIndirectMethod::Lumen
  ├─ IsScreenSpaceDiffuseIndirectSupported() → EDiffuseIndirectMethod::SSGI
  └─ ShouldRenderPluginGlobalIllumination()  → EDiffuseIndirectMethod::Plugin
```

**Lumen が選択される条件（`LumenDiffuseIndirect.cpp:291`）：**
- `r.Lumen.DiffuseIndirect.Allow=1`
- フィーチャーレベル SM5 以上
- `View.FinalPostProcessSettings.DynamicGlobalIlluminationMethod == Lumen`

---

## 非同期コンピュートパス（DispatchAsyncLumenIndirectLightingWork）

GBuffer 描画と **並列実行** するために、Lumen GI・反射の GPU 計算を  
AsyncCompute キューで先行発行する。

```
DispatchAsyncLumenIndirectLightingWork()     // IndirectLightRendering.cpp:889
  │
  ├─ LumenDiffuseIndirect::UseAsyncCompute() が true の場合のみ実行
  │
  ├─ [各ビュー]
  │   RenderLumenFinalGather(…, ERDGPassFlags::AsyncCompute)
  │     → Screen Probe Gather の結果を FSSDSignalTextures に書き込み
  │
  └─ [ReflectionsMethod == Lumen && bAsyncComputeReflections]
      RenderLumenReflections(…, ERDGPassFlags::AsyncCompute)
        → 反射テクスチャを FSSDSignalTextures::Textures[3] に書き込み
```

**結果は `FAsyncLumenIndirectLightingOutputs` に格納：**

```cpp
struct FAsyncLumenIndirectLightingOutputs
{
    TArray<FViewOutputs> ViewOutputs;  // ビューごとの出力
    ELumenIndirectLightingSteps StepsLeft;  // 残りの処理ステップ
};
```

`StepsLeft` は `ELumenIndirectLightingSteps` フラグで管理され、  
非同期完了時に `DoneAsync()` でフラグが下ろされる。

---

## 非同期ステップ管理（ELumenIndirectLightingSteps）

```cpp
enum class ELumenIndirectLightingSteps
{
    None             = 0,
    ScreenProbeGather = 1u << 0,  // Diffuse GI 計算
    Reflections       = 1u << 1,  // 反射計算
    Composite         = 1u << 3,  // 最終合成
    All               = ScreenProbeGather | Reflections | Composite
};
```

AsyncCompute が完了すると `StepsLeft` から `ScreenProbeGather`（と場合によって `Reflections`）が除去され、  
最終的に `Composite` のみが残った状態で `RenderDiffuseIndirectAndAmbientOcclusion()` が合成を行う。

---

## 最終合成パス（RenderDiffuseIndirectAndAmbientOcclusion）

```
RenderDiffuseIndirectAndAmbientOcclusion()           // IndirectLightRendering.cpp:977
  │
  ├─ [DiffuseIndirectMethod == Lumen && !bCompositeOnly]
  │   [Screen Probe Gather が未完の場合]
  │     RenderLumenFinalGather()                      // LumenScreenProbeGather.cpp:2094
  │     RenderLumenReflections()                      // LumenReflections.cpp
  │
  ├─ [AO 処理]
  │   RenderScreenSpaceAO() / RenderRayTracingAmbientOcclusion()
  │
  ├─ [デノイズ]
  │   IScreenSpaceDenoiser::DenoiseReflections()      // Lumen では Disabled
  │
  └─ [合成] ApplyDiffuseIndirect ラムダ               // IndirectLightRendering.cpp:1312
       ├─ FDiffuseIndirectCompositePS で描画
       │   → DiffuseIndirect_Lumen_[0-3] を GBuffer と合成 → SceneColor に加算
       └─ FAmbientCubemapCompositePS で描画（ある場合）
           → アンビエントキューブマップを加算
```

---

## FDiffuseIndirectCompositePS（合成シェーダー）

```cpp
class FDiffuseIndirectCompositePS : public FGlobalShader
{
    // --- シェーダーパーミュテーション ---
    class FApplyDiffuseIndirectDim   : SHADER_PERMUTATION_INT("DIM_APPLY_DIFFUSE_INDIRECT", 3);
    //   0 = なし（パスのみ）
    //   1 = SSGI / Plugin
    //   2 = Lumen（Screen Probe Gather）

    class FUpscaleDiffuseIndirectDim : SHADER_PERMUTATION_BOOL("DIM_UPSCALE_DIFFUSE_INDIRECT");
    //   SSGI のみ true

    class FScreenBentNormalMode      : SHADER_PERMUTATION_RANGE_INT("DIM_SCREEN_BENT_NORMAL_MODE", 0, 3);
    //   Lumen の Screen Bent Normal AO 強度モード

    class FShortRangeGI              : SHADER_PERMUTATION_BOOL("DIM_SHORT_RANGE_GI");
    //   Lumen の近距離 GI（ShortRangeAO 兼用）

    // --- 主要入力テクスチャ ---
    // DiffuseIndirect_Lumen_[0-3]: Lumen GI の4テクスチャ（Texture2DArray）
    //   [0] = Diffuse Indirect（メイン）
    //   [1] = Backface Diffuse
    //   [2] = Short Range GI
    //   [3] = Reflections
    //
    // AmbientOcclusionTexture: SSAO / RTAO マスク
    // PreIntegratedGF: BRDF の事前積分テーブル
    // SceneColorTexture: Dual Source Blending が使えない場合のコピー
};
// シェーダーファイル: /Engine/Private/DiffuseIndirectComposite.usf
```

---

## Substrate 対応（タイル描画）

UE5.3 以降で Substrate マテリアルシステムが有効の場合、  
`ApplyDiffuseIndirect` は **Substrate タイルタイプ別**（Simple / SingleLayer / Complex）に  
複数回呼ばれる。

```
ApplyDiffuseIndirect(ESubstrateTileType::Simple)
ApplyDiffuseIndirect(ESubstrateTileType::SingleLayerWater)
ApplyDiffuseIndirect(ESubstrateTileType::Complex)   // GGX 等の一般マテリアル
```

各タイルは `Substrate::FSubstrateTilePassVS` で分類済みのタイルマスクを使ってジオメトリを絞る。

---

## コード実行フロー

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─ CommitIndirectLightingState()                              // IndirectLightRendering.cpp:484
  │   └─ DiffuseIndirectMethod = Lumen / SSGI / Plugin を決定
  │
  ├─ DispatchAsyncLumenIndirectLightingWork()                   // IndirectLightRendering.cpp:889
  │   ├─ RenderLumenFinalGather(AsyncCompute)
  │   └─ RenderLumenReflections(AsyncCompute)
  │       ※ AsyncCompute キューで GBuffer 描画と並列実行
  │
  ├─ [GBuffer 描画 ...]
  │
  └─ RenderDiffuseIndirectAndAmbientOcclusion()                 // IndirectLightRendering.cpp:977
      ├─ [ScreenProbeGather が残っていれば] RenderLumenFinalGather()
      ├─ [Reflections が残っていれば]       RenderLumenReflections()
      ├─ [AO] RenderScreenSpaceAO()
      └─ ApplyDiffuseIndirect ラムダ × Substrate タイル数       // IndirectLightRendering.cpp:1312
          └─ FDiffuseIndirectCompositePS → SceneColor に加算書き込み
```

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|------------|--------|------|
| `FDeferredShadingSceneRenderer::CommitIndirectLightingState()` | `IndirectLightRendering.cpp:484` | 処理方式（Lumen/SSGI/Plugin）の決定 |
| `FDeferredShadingSceneRenderer::DispatchAsyncLumenIndirectLightingWork()` | `IndirectLightRendering.cpp:889` | AsyncCompute で GI・反射を先行実行 |
| `FDeferredShadingSceneRenderer::RenderDiffuseIndirectAndAmbientOcclusion()` | `IndirectLightRendering.cpp:977` | 最終合成のトップレベルエントリ |
| `FDiffuseIndirectCompositePS` | `IndirectLightRendering.cpp:119` | Diffuse + 反射を SceneColor に合成するシェーダー |
| `FAmbientCubemapCompositePS` | `IndirectLightRendering.cpp:231` | アンビエントキューブマップの加算シェーダー |
| `FAsyncLumenIndirectLightingOutputs` | `DeferredShadingRenderer.h:125` | AsyncCompute の出力テクスチャ管理 |
| `ELumenIndirectLightingSteps` | `DeferredShadingRenderer.h:115` | 残り処理ステップのフラグ管理 |
| `EDiffuseIndirectMethod` | `DeferredShadingRenderer.h:290` | 処理方式列挙（Disabled/Lumen/SSGI/Plugin）|

---

## 関連リファレンス

- [[ref_lumen_final_composite]] — `IndirectLightRendering.cpp` 主要クラス・関数リファレンス
