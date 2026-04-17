# a: Depth Pre-pass（RenderPrePass）

- 対象ファイル: `DepthRendering.h/.cpp` / `SceneRendering.cpp`
- 概要: [[13_depthprepass_overview]]

---

## 概要

`RenderPrePass()` は **不透明メッシュの深度だけを先行書き込みする EarlyZ パス**。  
後続の BasePass で DepthEqual テストにより不可視ピクセルの PS を早期棄却し、  
Overdraw によるシェーダー実行コストを削減する。

---

## 関数シグネチャ

```cpp
// DeferredShadingRenderer.h:645
void FDeferredShadingSceneRenderer::RenderPrePass(
    FRDGBuilder& GraphBuilder,
    TArrayView<FViewInfo> InViews,
    FRDGTextureRef SceneDepthTexture,     // 出力先深度テクスチャ
    FInstanceCullingManager& InstanceCullingManager,
    FRDGTextureRef* FirstStageDepthBuffer // オプション: 2段階深度バッファ
);
```

---

## EDepthDrawingMode とモード選択

```cpp
// DepthRendering.h:22
enum EDepthDrawingMode
{
    DDM_None            = 0, // PrePass 無効（Deferred を使わないプラットフォーム等）
    DDM_NonMaskedOnly   = 1, // Opaque のみ（Masked は BasePass で処理）
    DDM_AllOccluders    = 2, // Opaque + UseAsOccluder=true の Masked
    DDM_AllOpaque       = 3, // 全 Opaque（r.EarlyZPass=3 デフォルト）
    DDM_MaskedOnly      = 4, // Masked のみ
    DDM_AllOpaqueNoVelocity = 5, // Velocity パス分離のための Opaque 全体
};

// DepthRendering.cpp
FDepthPassInfo GetDepthPassInfo(const FScene* Scene)
{
    FDepthPassInfo Info;
    Info.EarlyZPassMode = (EDepthDrawingMode)CVarEarlyZPass.GetValueOnRenderThread();
    Info.bEarlyZPassMovable = CVarEarlyZPassMovable.GetValueOnRenderThread() != 0;
    // ...
    return Info;
}
```

---

## FDepthPassMeshProcessor

```cpp
// DepthRendering.h:181
class FDepthPassMeshProcessor : public FMeshPassProcessor
{
public:
    FDepthPassMeshProcessor(
        EMeshPass::Type InMeshPassType,
        const FScene* Scene,
        ERHIFeatureLevel::Type FeatureLevel,
        const FSceneView* InViewIfDynamicMeshCommand,
        const FMeshPassProcessorRenderState& InPassDrawRenderState,
        const bool InbRespectUseAsOccluderFlag, // UseAsOccluder フラグを尊重するか
        const EDepthDrawingMode InEarlyZPassMode,
        const bool InbEarlyZPassMovable,
        const bool bDitheredLODFadingOutMaskPass,
        FMeshPassDrawListContext* InDrawListContext,
        const bool bShadowProjection = false,
        const bool bSecondStageDepthPass = false);

    virtual void AddMeshBatch(...) override;
};
```

---

## TDepthOnlyVS / FDepthOnlyPS

```cpp
// DepthRendering.h:67
template <bool bUsePositionOnlyStream>
class TDepthOnlyVS : public FMeshMaterialShader
{
    // bUsePositionOnlyStream=true: 座標のみのストリームを使用（帯域節約）
    // → Opaque メッシュ（UV 不要）に使用
    // bUsePositionOnlyStream=false: 通常の頂点ストリーム
    // → Masked / PDO（Pixel Depth Offset）付きマテリアルに使用
};

class FDepthOnlyPS : public FMeshMaterialShader
{
    // Masked マテリアルのみコンパイル
    // OpacityMask < ClipValue の場合 discard
    // SLW（Single Layer Water）は SceneDepth 参照のため例外
};
```

---

## RenderPrePass() 内部フロー

```
RenderPrePass()                                 DepthRendering.cpp
  │
  ├─ [1] RenderPass 開始
  │   SceneDepthTexture（D24S8）を DSV にバインド
  │   LoadAction: r.ClearSceneMethod に応じて Clear or Load
  │
  ├─ [2] Dithered LOD Stencil（有効時）
  │   AddDitheredStencilFillPass()
  │   → Stencil を LOD フェード用マスクで書き込み
  │
  ├─ [3] Nanite Depth Emit
  │   Nanite::EmitDepth() or Nanite::EmitSceneDepthStencil()
  │   → VisBuffer（ClusterID + TriangleID）から Compute Shader で
  │     深度値を SceneDepth テクスチャに直接書き込み
  │
  ├─ [4] 非 Nanite Opaque
  │   [r.ParallelPrePass=1] FParallelCommandListSet で並列
  │   DrawDynamicMeshPass(View, FDepthPassMeshProcessor, ...)
  │     AddMeshBatch():
  │       bUsePositionOnlyStream=true → TDepthOnlyVS<true>
  │       PS なし（Depth Only）
  │       → RHIDrawIndexedPrimitive（深度値のみ書き込み）
  │
  └─ [5] 非 Nanite Masked
      DrawDynamicMeshPass(View, FDepthPassMeshProcessor(DDM_MaskedOnly), ...)
        AddMeshBatch():
          bUsePositionOnlyStream=false → TDepthOnlyVS<false> + FDepthOnlyPS
          PS で OpacityMask サンプル → clip() で早期棄却

【BasePass との関係】
  PrePass 完了後、BasePass は DepthStencilState を DepthEqual に変更
  → 既に書き込まれた深度と一致するピクセルのみ PS を実行
  → Overdraw（重なり部分の無駄な PS 実行）を排除
```

---

## 関連リファレンス

- [[ref_depth_rendering]] — `TDepthOnlyVS` / `FDepthOnlyPS` / `FDepthPassMeshProcessor` 詳細
- [[b_velocity_rendering]] — Velocity Buffer パス（同タイミングで描画）
