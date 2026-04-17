# a: RenderLights() フロー

- 対象ファイル: `LightRendering.h/.cpp` / `DeferredLightRendering.h/.cpp`
- 概要: [[16_deferred_lighting_overview]]

---

## 概要

`RenderLights()` は `FSortedLightSetSceneInfo` を受け取り、  
ライト種別・シャドウの有無に応じて **Clustered / Standard / UnbatchedLights** の  
3経路に振り分けて直接照明を SceneColor に書き込む。

---

## 関数シグネチャ

```cpp
// LightRendering.cpp:1520
void FDeferredShadingSceneRenderer::RenderLights(
    FRDGBuilder& GraphBuilder,
    FMinimalSceneTextures& SceneTextures,   // GBuffer テクスチャ群
    FRDGTextureRef LightingChannelsTexture, // ライティングチャンネルマスク
    const FSortedLightSetSceneInfo& SortedLightSet  // ソート済みライト一覧
);
```

---

## コード実行フロー

```
RenderLights()                                    LightRendering.cpp:1520
  │
  ├─ [Substrate有効] AddSubstrateStencilPass()
  │   → GBuffer 後のステンシルを Fast/Single/Complex に更新
  │
  ├─ ShadowSceneRenderer.RenderVirtualShadowMapProjectionMaskBits()
  │   → VSM Shadow Mask Bits テクスチャを生成（後で各ライトで参照）
  │
  ├─ ─── BatchedLights ──────────────────────────────────────────
  │   │
  │   ├─ [ClusteredDeferred有効 && LightGridに注入済み]
  │   │   AddClusteredDeferredShadingPass()      ClusteredDeferredShadingRenderer.cpp
  │   │   → SimpleLights + ClusteredSupported を CS 一括処理
  │   │   → StandardDeferredStart = ClusteredSupportedEnd にずらす
  │   │
  │   └─ [Clustered非使用] RenderSimpleLightsStandardDeferred()
  │       → パーティクルライト等を Standard Deferred で処理
  │
  ├─ ─── Standard Deferred（影なし）──────────────────────────────
  │   └─ for(LightIndex in [StandardDeferredStart, UnbatchedLightStart))
  │       └─ RenderLight(LightSceneInfo, Shadow=nullptr, ...)
  │           → 球/コーン境界ジオメトリを描画
  │           → DeferredLightPixelShaders.usf#MainPS() でシェーディング
  │
  └─ ─── UnbatchedLights（影あり）────────────────────────────────
      └─ for(LightIndex in [UnbatchedLightStart, end))
          │
          ├─ [RT Shadow有効] RayTracedShadow / MegaLights RT シャドウ
          ├─ [従来Shadow] RenderProjectedShadow()
          │   → FProjectedShadowInfo の Shadow Mask 生成
          │
          ├─ [Denoiser有効] IScreenSpaceDenoiser.DenoiseShadows()
          │
          └─ RenderLight(LightSceneInfo, ShadowMaskTexture, ...)
```

---

## RenderLight() 内部

```cpp
// LightRendering.cpp:2667 付近
// 境界ジオメトリを使って PS の実行範囲を絞る
void RenderLight(
    FRDGBuilder& GraphBuilder,
    FScene* Scene,
    const FViewInfo& View,
    FMinimalSceneTextures& SceneTextures,
    const FLightSceneInfo* LightSceneInfo,
    FRDGTextureRef ScreenShadowMaskTexture,   // Shadow Mask（null=影なし）
    FRDGTextureRef LightingChannelsTexture,
    bool bRenderOverlap,
    bool bCloudShadow,
    bool bUseLightFunctionAtlas,
    ...,
    ESubstrateTileType SubstrateTileType)     // Substrate タイル種別
{
    // ライト種別ごとの境界ジオメトリ
    // Point Light  → TStencilSphereVertexBuffer（球）
    // Spot Light   → DrawCone（コーン + 球面キャップ）
    // Directional  → DrawRectangle（全画面クワッド）
    // Rect Light   → DrawCone（近似錐台）
}
```

---

## ライト種別と境界ジオメトリ

| ライト種別 | 境界ジオメトリ | ステンシル利用 |
|----------|-------------|------------|
| Point Light | 球（TStencilSphereVertexBuffer）| ○（前面→ステンシル書き / 後面→シェーディング）|
| Spot Light | コーン + 球面キャップ（DrawCone）| ○ |
| Directional Light | 全画面クワッド | × |
| Rect Light | コーン近似 | ○ |

---

## 関連リファレンス

- [[ref_light_rendering]] — `FDeferredLightVS` / `FDeferredLightPS` / `FLightSceneInfo`
- [[ref_light_params]] — `FDeferredLightUniformStruct` フィールド一覧
- [[b_clustered_tiled]] — Clustered Deferred 詳細

---

## ライト種別ごとの RenderLight() ディスパッチ詳細フロー

```
RenderLight()                                   LightRendering.cpp
  │
  ├─ [共通] FDeferredLightUniformStruct 構築
  │   GetDeferredLightParameters(View, *LightSceneInfo)
  │     → ShadowMapChannelMask / ContactShadow / LightingChannelMask を詰める
  │   SetDeferredLightParameters(BatchedParams, DeferredLightUniformBuffer, ...)
  │
  ├─ [ライト種別分岐]
  │
  │   ── Directional Light ──────────────────────────────────────
  │   FDeferredLightVS(FRadialLight=false)
  │     → FullScreenRect（4頂点クワッド）を出力
  │   FDeferredLightPS(SHADER_RADIAL_LIGHT=0)
  │     → 全ピクセルで GBuffer デコード + BRDF 評価
  │     → ContactShadow をレイマーチングで計算（有効時）
  │     → SceneColor に加算
  │
  │   ── Point Light ──────────────────────────────────────────
  │   FDeferredLightVS(FRadialLight=true)
  │     → LightBounds（球ジオメトリ）の頂点を変換
  │   ステンシルパス（2パス描画）:
  │     [Pass 1] Front Face → ステンシル書き込み（カウントアップ）
  │     [Pass 2] Back Face  → ステンシルテストOKのピクセルのみシェーディング
  │   FDeferredLightPS(SHADER_RADIAL_LIGHT=1)
  │     → ライスト境界内のピクセルのみ BRDF 評価
  │
  │   ── Spot Light ──────────────────────────────────────────
  │   上記 Point Light と同様だがコーン形状ジオメトリを使用
  │   SpotAngle の cos 値で照射範囲を追加フィルタリング
  │
  │   ── Rect Light ──────────────────────────────────────────
  │   LTC（Linearly Transformed Cosines）でエリアライトのBRDFを近似
  │   コーン近似ジオメトリ + LTC テーブルのサンプリング
  │
  ├─ [Shadow Mask 統合]
  │   ScreenShadowMaskTexture != null の場合:
  │     DeferredLightPixelShaders.usf にバインド
  │     → Shadow Mask テクスチャの対応チャンネルをサンプル
  │     → ライスト寄与に ShadowFactor を乗算
  │
  └─ [Light Function 統合]
      bUseLightFunctionAtlas=true の場合:
        LightFunctionAtlas テクスチャをサンプル
      bUseLightFunctionAtlas=false かつ LF あり:
        RenderLightFunction() で専用テクスチャに描画してからバインド

[Substrate タイルループ]
  Substrate 有効時は for each SubstrateTileType（Fast/Single/Complex）:
    ステンシルマスクで描画エリアを制限して同一ライストを再度 RenderLight()
```

> [!note]- Depth Bounds Test
> `r.AllowDepthBoundsTest=1`（デフォルト）の場合、ライスト球の Z 範囲を
> DepthBoundsTest として RHI に設定する。
> ライスト球より手前・奥のピクセルをハードウェアで早期棄却でき、
> GPU ALU コストを削減できる。
