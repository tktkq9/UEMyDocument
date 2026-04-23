# Deferred Lighting 全体概要

- 取得日: 2026-04-12
- 対象: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\LightRendering.h/.cpp`
- 上位: [[01_rendering_overview]]
- Details: [[a_light_rendering]] | [[b_clustered_tiled]] | [[c_reflection_skylight]] | [[d_light_functions]]
- Reference: [[ref_light_rendering]] | [[ref_clustered_tiled]] | [[ref_light_params]]
- GPU 対応: [[GPU/DeferredLighting/TASK_CHECKLIST]]

---

## Deferred Lighting とは

**GBuffer を読み取り、全ライトの直接照明を SceneColor に加算するパス**。  
BasePass で GBuffer に格納されたマテリアル情報（BaseColor / Normal / Roughness 等）と  
各ライトのパラメータを組み合わせて BRDF 評価を行う。

| ライト処理方式 | 対象 | 説明 |
|------------|------|------|
| Clustered Deferred | 多数のローカルライト | 3D クラスターグリッドにライトを割り当て CS で一括処理 |
| Tiled Deferred | ローカルライト（フォールバック）| 画面を 2D タイルに分割して CS で処理 |
| Standard Deferred | UnbatchedLights | ライト境界ジオメトリ（球・コーン）を描画して PS で処理 |
| MegaLights | 多数のダイナミックライト | 確率的サンプリング（UE5.4+）|

---

## 全体アーキテクチャ

```mermaid
graph TD
    subgraph Sort["ライトソート（GatherLights）"]
        SL[FSortedLightSetSceneInfo<br>SimpleLights / BatchedLights / UnbatchedLights<br>/ MegaLights に分類]
    end

    subgraph Clustered["Clustered/Tiled Deferred"]
        CL[AddClusteredDeferredShadingPass<br>FClusteredDeferredLightingCS]
        TL[RenderSimpleLightsStandardDeferred<br>単純ライト用]
    end

    subgraph Standard["Standard Deferred（UnbatchedLights）"]
        UB[RenderLight() × UnbatchedLights<br>境界ジオメトリ描画 → PS でシェーディング]
    end

    subgraph Shadow["シャドウ統合"]
        VSM[VSM Projection<br>ShadowSceneRenderer.RenderVirtualShadowMapProjectionMaskBits]
        OtherShadow[従来 Shadow Projection<br>FProjectedShadowInfo]
    end

    subgraph Output["SceneColor（加算書き込み）"]
    end

    Sort --> Clustered & Standard
    Shadow --> Standard
    Clustered --> Output
    Standard --> Output
```

---

## フレームの流れ（概略）

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─ GatherLightsForView()                    ライトの可視性・分類
  │   └─ FSortedLightSetSceneInfo を構築
  │       ├─ SimpleLightsEnd         : パーティクルライト等の単純ライト
  │       ├─ ClusteredSupportedEnd   : Clustered Deferred 対象ライト
  │       ├─ UnbatchedLightStart     : Standard Deferred ライト
  │       └─ MegaLightsLightStart    : MegaLights 対象ライト
  │
  └─ RenderLights()                           LightRendering.cpp:1520
      │
      ├─ [Substrate有効] AddSubstrateStencilPass()
      ├─ ShadowSceneRenderer.RenderVirtualShadowMapProjectionMaskBits()
      │
      ├─ [Clustered有効] AddClusteredDeferredShadingPass()
      │   └─ Simple + Clustered 対応ライトを CS 一括処理
      │
      ├─ RenderSimpleLightsStandardDeferred() （Clustered 非使用時）
      │
      ├─ [StandardDeferredStart → UnbatchedLightStart]
      │   └─ RenderLight() × 各ライト （Substrate 有効時は TileType 外ループ）
      │
      └─ [UnbatchedLightStart → end]          シャドウ付きライト
          ├─ RT Shadow（DXR） または 従来 Shadow Projection
          ├─ デノイズ（IScreenSpaceDenoiser）
          └─ RenderLight() + Shadow Mask
```

---

## `FSortedLightSetSceneInfo` のインデックス境界

```cpp
struct FSortedLightSetSceneInfo
{
    // [0 .. SimpleLightsEnd)           → SimpleLights（パーティクルライト等）
    // [SimpleLightsEnd .. ClusteredEnd)→ Clustered Deferred 処理
    // [ClusteredEnd .. UnbatchedStart) → Standard Deferred（影なし）
    // [UnbatchedStart .. end)          → Standard Deferred（影あり）
    // MegaLightsLightStart             → MegaLights 対象の開始インデックス
    int32 SimpleLightsEnd;
    int32 ClusteredSupportedEnd;
    int32 UnbatchedLightStart;
    int32 MegaLightsLightStart;
    FSimpleLightArray SimpleLights;
    TArray<FSortedLightSceneInfo> SortedLights;
};
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.AllowDepthBoundsTest` | 1 | 深度バウンズテスト（ライト境界内のピクセルのみ処理）|
| `r.ClusteredDeferredShading` | 1 | Clustered Deferred Shading の有効/無効 |
| `r.AllowSimpleLights` | 1 | パーティクルライト等の SimpleLights 処理 |
| `r.Shadow.Denoiser` | 1 | シャドウデノイザーの使用 |

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `LightRendering.h/.cpp` | `RenderLights()` メインオーケストレーション |
| `DeferredLightRendering.h/.cpp` | `RenderLight()` 各ライト描画 |
| `ClusteredDeferredShadingRenderer.h/.cpp` | Clustered Deferred Shading パス |
| `LightFunctionRendering.cpp` | Light Function マテリアル適用 |
| `Shadows/ShadowSceneRenderer.h/.cpp` | Shadow シーン管理 |
| `LightSceneInfo.h` | `FLightSceneInfo` / `FSortedLightSceneInfo` |

---

## RenderLights() → ライト分類 → Clustered/Direct → Shadow Projection 詳細フロー

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─ [事前] GatherLightsAndComputeLightGrid()
  │   │
  │   ├─ for each FLightSceneInfo in Scene->Lights:
  │   │   GetLightOcclusionType() → Shadowmap / Raytraced / MegaLights / MegaLightsVSM
  │   │   SortKey にライストタイプ・シャドウ有無・コストを詰める
  │   │
  │   ├─ std::sort(SortedLights, SortKey)
  │   │   → SimpleLightsEnd / ClusteredSupportedEnd / UnbatchedLightStart / MegaLightsStart を確定
  │   │
  │   └─ InjectLightsIntoGrid()（Clustered Deferred 用）
  │       → FForwardLightData 変換 → LightDataBuffer / ClusteredLightGrid を構築
  │
  └─ RenderLights()                               LightRendering.cpp:1520
      │
      ├─ [A] Substrate ステンシルパス（Substrate有効時）
      │   AddSubstrateStencilPass()
      │   → GBuffer のステンシルビットを Fast/Single/Complex に更新
      │
      ├─ [B] Shadow Projection 前処理
      │   ShadowSceneRenderer.RenderVirtualShadowMapProjectionMaskBits()
      │   → MegaLights VSM / Clustered Deferred 用の ShadowMaskBits テクスチャを生成
      │   FProjectedShadowInfo::RenderProjectedShadow()（UnbatchedLights 用）
      │   → CSM / Stationary ShadowMap を描画
      │
      ├─ [C] Clustered Deferred（バッチライト）
      │   if ShouldUseClusteredDeferredShading() && AreLightsInLightGrid():
      │     AddClusteredDeferredShadingPass()
      │       → [0 .. ClusteredSupportedEnd) を CS 一括処理
      │       → StandardDeferredStart = ClusteredSupportedEnd にシフト
      │   else:
      │     RenderSimpleLightsStandardDeferred()
      │       → [0 .. SimpleLightsEnd) を Standard Deferred で処理
      │
      ├─ [D] Standard Deferred（影なしライト）
      │   for LightIdx in [StandardDeferredStart .. UnbatchedLightStart):
      │     RenderLight(LightSceneInfo, Shadow=nullptr, SubstrateTileType)
      │       → 境界ジオメトリ（球/コーン/全画面クワッド）描画
      │       → DeferredLightPixelShaders.usf でシェーディング → SceneColor 加算
      │
      └─ [E] UnbatchedLights（影付きライト）
          for LightIdx in [UnbatchedLightStart .. end):
            │
            ├─ [RT Shadow] RayTracedShadow / MegaLights RT
            │   → TLAS へシャドウレイ → ShadowMask テクスチャ生成
            │
            ├─ [従来Shadow] Shadow Mask 適用
            │   → FProjectedShadowInfo から ShadowMaskTexture を読み込み
            │
            ├─ [Denoiser] IScreenSpaceDenoiser.DenoiseShadows()
            │   → テンポラル蓄積でシャドウノイズ除去
            │
            └─ RenderLight(LightSceneInfo, ShadowMaskTexture, ...)
                → 境界ジオメトリ描画 + Shadow Mask 乗算 → SceneColor 加算
```
