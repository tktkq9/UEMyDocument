# ref: TDepthOnlyVS / FDepthOnlyPS / FDepthPassMeshProcessor

- 対象ファイル: `DepthRendering.h/.cpp`
- 概要: [[13_depthprepass_overview]]

---

## EDepthDrawingMode（DepthRendering.h:22）

```cpp
enum EDepthDrawingMode
{
    DDM_None              = 0, // Depth Pre-pass 無効（上位で判定済み）
    DDM_NonMaskedOnly     = 1, // Opaque のみ（Masked はスキップ）
    DDM_AllOccluders      = 2, // Opaque + Masked（bUseAsOccluder=false はスキップ）
    DDM_AllOpaque         = 3, // 全 Opaque（Masked 含む・全ピクセル一致必須）
    DDM_MaskedOnly        = 4, // Masked のみ
    DDM_AllOpaqueNoVelocity = 5, // 全 Opaque（Velocity は別パスで処理）
};
```

---

## FDepthPassInfo（DepthRendering.h:38）

```cpp
struct FDepthPassInfo
{
    EDepthDrawingMode EarlyZPassMode = DDM_None;  // 上記モードから選択
    bool bEarlyZPassMovable = false;               // 動的メッシュもDepthPassに含めるか
    bool bDitheredLODTransitionsUseStencil = false; // LOD ディザにステンシル使用
    ERDGPassFlags StencilDitherPassFlags = ERDGPassFlags::Raster;

    bool IsComputeStencilDitherEnabled() const;    // CS ベースのステンシルディザか
    bool IsRasterStencilDitherEnabled() const;     // ラスタ ベースのステンシルディザか
};

// 取得方法
FDepthPassInfo GetDepthPassInfo(const FScene* Scene);
// → r.EarlyZPass / Scene のシャドウ設定 / HW Occlusion Culling フラグから決定
```

---

## TDepthOnlyVS（DepthRendering.h:66）

```cpp
// テンプレートパラメータ: bUsePositionOnlyStream
template <bool bUsePositionOnlyStream>
class TDepthOnlyVS : public FMeshMaterialShader
{
    DECLARE_SHADER_TYPE(TDepthOnlyVS, MeshMaterial);

    static bool ShouldCompilePermutation(
        const FMeshMaterialShaderPermutationParameters& Parameters)
    {
        if (bUsePositionOnlyStream)
        {
            // Position-Only ストリーム: LocalVertexFactory のみ、SpecialEngineMaterial のみ
            return Parameters.VertexFactoryType->SupportsPositionOnly()
                && Parameters.MaterialParameters.bIsSpecialEngineMaterial;
        }

        if (IsTranslucentBlendMode(Parameters.MaterialParameters))
        {
            // 半透明だが CustomDepth を書く場合のみ
            return Parameters.MaterialParameters.bIsTranslucencyWritingCustomDepth;
        }

        // Opaque の通常 Depth Only VS:
        // SpecialEngineMaterial OR Masked (bWritesEveryPixel=false) OR
        // MeshPositionOffset あり → コンパイル
        // Nanite はスキップ（bSupportsNaniteRendering）
        return (Parameters.MaterialParameters.bIsSpecialEngineMaterial
            || !Parameters.MaterialParameters.bWritesEveryPixel
            || Parameters.MaterialParameters.bMaterialMayModifyMeshPosition)
            && !Parameters.VertexFactoryType->SupportsNaniteRendering();
    }

    // ─── bUsePositionOnlyStream=true（Opaque 向け）────────────────
    // Position Attribute のみのストリームを使用
    // → UV / Normal / Tangent を送らないため頂点バンド幅を節約
    // → LocalVertexFactory::SupportsPositionOnly() が必要

    // ─── bUsePositionOnlyStream=false（Masked / CustomDepth 向け）──
    // フル頂点ストリームを使用
    // → UV が必要（OpacityMask テクスチャサンプリングのため）
};
```

---

## FDepthOnlyPS（DepthRendering.h:122）

```cpp
class FDepthOnlyPS : public FMeshMaterialShader
{
    DECLARE_SHADER_TYPE(FDepthOnlyPS, MeshMaterial);

    static bool ShouldCompilePermutation(
        const FMeshMaterialShaderPermutationParameters& Parameters)
    {
        if (IsTranslucentBlendMode(Parameters.MaterialParameters))
        {
            return Parameters.MaterialParameters.bIsTranslucencyWritingCustomDepth;
        }

        // Masked マテリアルまたは Pixel Depth Offset がある場合のみ必要
        return (!Parameters.MaterialParameters.bWritesEveryPixel
             || Parameters.MaterialParameters.bHasPixelDepthOffsetConnected)
            && !Parameters.VertexFactoryType->SupportsNaniteRendering();
    }

    static void ModifyCompilationEnvironment(...)
    {
        // SingleLayerWater は SceneTextures を使えるが通常は SCENE_TEXTURES_DISABLED=1
        // AllowDebugViewModes の設定
    }

    // ─── 処理内容 ─────────────────────────────────────────────────
    // Masked: OpacityMask テクスチャサンプル → ClipValue より低ければ discard
    // PDO (Pixel Depth Offset): PDO 値で SV_Depth を調整して書き込み
    //
    // Opaque で PDO なし: PS 不要（深度バッファ直書き）
    // → AddMeshBatch でピクセルシェーダーを nullptr に設定
};
```

---

## FDepthPassMeshProcessor（DepthRendering.h:181）

```cpp
class FDepthPassMeshProcessor
    : public FSceneRenderingAllocatorObject<FDepthPassMeshProcessor>
    , public FMeshPassProcessor
{
public:
    FDepthPassMeshProcessor(
        EMeshPass::Type InMeshPassType,
        const FScene* Scene,
        ERHIFeatureLevel::Type FeatureLevel,
        const FSceneView* InViewIfDynamicMeshCommand,
        const FMeshPassProcessorRenderState& InPassDrawRenderState,
        const bool InbRespectUseAsOccluderFlag,  // bUseAsOccluder フラグを尊重するか
        const EDepthDrawingMode InEarlyZPassMode, // どのメッシュを描画するか
        const bool InbEarlyZPassMovable,           // 動的メッシュを含めるか
        const bool bDitheredLODFadingOutMaskPass,  // LOD フェードマスクパスか
        FMeshPassDrawListContext* InDrawListContext,
        const bool bShadowProjection = false,      // Shadow Projection 用に流用するか
        const bool bSecondStageDepthPass = false); // 二段階 Depth Pass か

    virtual void AddMeshBatch(...) override final;

    // ─── AddMeshBatch の処理フロー ────────────────────────────────
    // 1. EarlyZPassMode に応じてメッシュを受け入れるか判定
    //    DDM_NonMaskedOnly: Masked は拒否
    //    DDM_MaskedOnly:    Opaque は拒否
    //    DDM_AllOccluders:  bUseAsOccluder=false なら拒否
    //
    // 2. Opaque（bWritesEveryPixel=true, PDO なし）:
    //    TDepthOnlyVS<true>  (Position-Only)
    //    PixelShader = nullptr
    //
    // 3. Masked（bWritesEveryPixel=false）または PDO あり:
    //    TDepthOnlyVS<false> (Full Stream)
    //    FDepthOnlyPS
    //
    // 4. BuildMeshDrawCommands → Submit
};
```

---

## GetDepthPassShaders() ヘルパー

```cpp
// DepthRendering.h:172
template <bool bPositionOnly>
bool GetDepthPassShaders(
    const FMaterial& Material,
    const FVertexFactoryType* VertexFactoryType,
    ERHIFeatureLevel::Type FeatureLevel,
    bool bMaterialUsesPixelDepthOffset,
    TShaderRef<TDepthOnlyVS<bPositionOnly>>& VertexShader,
    TShaderRef<FDepthOnlyPS>& PixelShader,
    FShaderPipelineRef& ShaderPipeline);
// → Material の特性に基づき適切なシェーダーペアを取得
```

---

## Dithered LOD ステンシル

```
AddDitheredStencilFillPass()    DepthRendering.h:58
  → LOD フェードアウト中ピクセルをステンシルバッファでマーク
  → EarlyZ Pass で discard の代わりにステンシル判定に使用

AddDitheredStencilClearPass()
  → フレーム末尾でステンシルをクリア
```

---

## 関連リファレンス

- [[ref_velocity_rendering]] — `FVelocityMeshProcessor` / Velocity パス
- [[a_depth_rendering]] — RenderPrePass() 詳細フロー
