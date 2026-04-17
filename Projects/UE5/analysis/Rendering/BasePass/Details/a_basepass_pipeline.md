# a: Base Pass パイプライン

- 対象ファイル: `BasePassRendering.h/.cpp`
- 概要: [[14_basepass_overview]]

---

## 概要

`RenderBasePass()` は **不透明メッシュを GBuffer に書き込む** メインパス。  
並列コマンドリスト（`r.ParallelBasePass=1`）で各ビューを並列描画する。  
Nanite メッシュは通常の Draw Call ではなく `EmitGBuffer()` の Compute シェーダーで GBuffer に書き込む。

---

## 関数シグネチャ

```cpp
// BasePassRendering.cpp:1071
// static メソッド: Custom Render Pass 対応のため ViewFamily に依存しない設計
static void FDeferredShadingSceneRenderer::RenderBasePass(
    FDeferredShadingSceneRenderer& Renderer,
    FRDGBuilder& GraphBuilder,
    TArrayView<FViewInfo> InViews,
    FSceneTextures& SceneTextures,
    const FDBufferTextures& DBufferTextures,    // Deferred Decal DBuffer
    FExclusiveDepthStencil::Type BasePassDepthStencilAccess,
    FRDGTextureRef ForwardShadowMaskTexture,    // Forward Shading 用
    FInstanceCullingManager& InstanceCullingManager,
    bool bNaniteEnabled,
    FNaniteShadingCommands& NaniteBasePassShadingCommands,
    const TArrayView<Nanite::FRasterResults>& NaniteRasterResults
);
```

---

## コード実行フロー

```
RenderBasePass()                              BasePassRendering.cpp:1071
  │
  ├─ GBuffer クリア方法を決定
  │   r.ClearSceneMethod: 0=なし / 1=RHIClear / 2=Far-Z Quad
  │   Wireframe / ShaderComplexity → 常に RHIClear
  │
  ├─ bDoParallelBasePass = GRHICommandList.UseParallelAlgorithms()
  │                      && CVarParallelBasePass
  │                      && !bDebugViewMode && !bRenderLightmapDensity
  │
  ├─ SceneTextures.GetGBufferRenderTargets(BasePassTextures)
  │   → GBufferA/B/C/D/E/F + SceneColor + Velocity を RT 配列に束ねる
  │
  ├─ [bNaniteEnabled] Nanite::EmitGBuffer()
  │   → VisBuffer → GBuffer への Compute シェーダー変換
  │   → NaniteBasePassShadingCommands を CS でディスパッチ
  │
  └─ RenderBasePassInternal()                 BasePassRendering.cpp:1449
      │
      ├─ RDG RenderPass 開始（GBuffer RT バインド）
      │
      ├─ [bDoParallelBasePass]
      │   FParallelCommandListSet を生成
      │   各ビューごとに ParallelCommandList で DrawCall を並列実行
      │
      └─ [通常] 各ビューを逐次描画
          └─ DrawDynamicMeshPass() / DrawDynamicMeshPassWithEarlyExit()
              └─ TBasePassMeshProcessor::AddMeshBatch()
                  → マテリアル × ライトマップ → シェーダー排列確定
                  → FMeshDrawCommand 生成・Submit
```

---

## GBuffer RT 束ね処理

```cpp
// FSceneTextures::GetGBufferRenderTargets()
// 返す RT スロット順:
// [0] SceneColor
// [1] GBufferA  (WorldNormal)
// [2] GBufferB  (Metallic/Spec/Rough/ShadingModelID)
// [3] GBufferC  (BaseColor + AO)
// [4] GBufferD  (CustomData)
// [5] GBufferE  (PreShadow)
// [6] GBufferF  (Tangent/Anisotropy、有効時のみ)
// [7] Velocity  (Motion Vector)
uint32 FSceneTextures::GetGBufferRenderTargets(
    TStaticArray<FTextureRenderTargetBinding, MaxSimultaneousRenderTargets>& OutRTs) const;
```

---

## Nanite との分担

| 処理 | 担当 | フロー |
|-----|------|--------|
| Nanite メッシュの可視クラスター判定 | Nanite CullRaster | VisBuffer（uint64）生成 |
| VisBuffer → GBuffer 変換 | Nanite EmitGBuffer CS | `NaniteShadingCommands` を CS でディスパッチ |
| 非 Nanite メッシュ | TBasePassMeshProcessor | 通常の VS+PS DrawCall |

---

## 関連リファレンス

- [[ref_basepass_renderer]] — `TBasePassMeshProcessor` / `FBasePassVS` / `FBasePassPS`
- [[ref_basepass_common]] — `FOpaqueBasePassUniformParameters`
- [[ref_gbuffer_textures]] — `FSceneTextures` GBuffer テクスチャ

---

## SetupBasePassState() → DispatchDraw() → Nanite GBuffer Resolve 詳細フロー

```
[Step 1] SetupBasePassState()
  │ BasePassRendering.cpp
  │
  ├─ DepthStencil アクセスモード設定
  │   bDepthWriteEnabled: EarlyZ パス済み → DepthRead のみ
  │   例外: HitProxy / Wireframe では DepthWrite も ON
  │
  ├─ BlendState 設定
  │   不透明: No Blend（上書き）
  │   半透明BasePass: Translucent Blend（アルファブレンド）
  │
  └─ RenderState → FMeshPassProcessorRenderState に格納

[Step 2] DispatchDraw（非 Nanite）
  │
  ├─ DrawDynamicMeshPass()
  │   → TBasePassMeshProcessor::AddMeshBatch() ループ
  │
  └─ FParallelCommandListSet::ExecuteCommands()
      → RHI レベルの DrawIndexedPrimitive / DrawIndexedPrimitiveIndirect

[Step 3] Nanite GBuffer Resolve
  │
  ├─ NaniteShading::EmitGBuffer()
  │   入力: FNaniteRasterResults（VisBuffer = Texture2D<uint64>）
  │
  ├─ TileClassify CS
  │   → VisBuffer の ClusterID → ShadingBin へのマッピング
  │
  ├─ ShadeBinning CS（FMaterial × ShadingBin ごと）
  │   → NaniteBasePassShadingCommands を1 bin = 1 CS Dispatch で実行
  │   → BasePassVertexShader.usf / BasePassPixelShader.usf に相当する処理を
  │     Compute Shader で代替
  │
  └─ GBuffer RT に書き込み完了
      → 以降の Lighting Pass は非 Nanite と同じ GBuffer を参照

[タイミング関係]
  Nanite EmitGBuffer  ─→  RenderBasePassInternal（非Nanite）
        ↓                          ↓
  共通 GBuffer RT 共有（書き込みは排他的タイル分割）
```

> [!note]- EarlyZ との関係
> DepthPrePass（EarlyZ）が実行済みの場合、BasePass の DepthStencil は
> `EDepthDrawingMode::DM_None` または `DM_Opaque`（Masked のみ）になり
> 深度書き込みをスキップして帯域を節約する。
> `r.EarlyZPass=3`（デフォルト）で Opaque + Masked の両方に EarlyZ が適用される。
