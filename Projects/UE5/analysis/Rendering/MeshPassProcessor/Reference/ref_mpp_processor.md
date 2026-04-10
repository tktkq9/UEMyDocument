# REF: FMeshPassProcessor

- 対象ファイル: `MeshPassProcessor.h`, `MeshPassProcessor.inl`, `MeshPassProcessor.cpp`
- 関連Details: [[a_mpp_pipeline]]

---

## FMeshPassProcessor — 基底クラス

```cpp
class FMeshPassProcessor : public IPSOCollector
{
public:
    // ─── コンテキスト ───
    EMeshPass::Type           MeshPassType;
    const FScene*             Scene;
    ERHIFeatureLevel::Type    FeatureLevel;
    const FSceneView*         ViewIfDynamicMeshCommand;  // 動的のみ有効
    FMeshPassDrawListContext*  DrawListContext;

    // ─── コンストラクタ ───
    FMeshPassProcessor(
        EMeshPass::Type InMeshPassType,
        const FScene* InScene,
        ERHIFeatureLevel::Type InFeatureLevel,
        const FSceneView* InViewIfDynamicMeshCommand,
        FMeshPassDrawListContext* InDrawListContext);

    virtual ~FMeshPassProcessor() {}

    // ─── 純粋仮想（サブクラスが必ず実装）───
    virtual void AddMeshBatch(
        const FMeshBatch& MeshBatch,
        uint64 BatchElementMask,
        const FPrimitiveSceneProxy* PrimitiveSceneProxy,
        int32 StaticMeshId = -1) = 0;

    // ─── コアメソッド（基底クラス提供）───
    template<typename PassShadersType, typename ShaderElementDataType>
    void BuildMeshDrawCommands(
        const FMeshBatch& MeshBatch,
        uint64 BatchElementMask,
        const FPrimitiveSceneProxy* PrimitiveSceneProxy,
        const FMaterialRenderProxy& MaterialRenderProxy,
        const FMaterial& MaterialResource,
        const FMeshPassProcessorRenderState& DrawRenderState,
        const PassShadersType& PassShaders,
        ERasterizerFillMode MeshFillMode,
        ERasterizerCullMode MeshCullMode,
        FMeshDrawCommandSortKey SortKey,
        EMeshPassFeatures MeshPassFeatures,
        const ShaderElementDataType& ShaderElementData);

    // ─── 静的ユーティリティ ───
    static ERasterizerCullMode InverseCullMode(ERasterizerCullMode CullMode);
    static ERasterizerFillMode ComputeMeshFillMode(
        const FMaterial& InMaterialResource,
        const FMeshBatch& InMeshBatch,
        bool bForceWireframe = false);
    static ERasterizerCullMode ComputeMeshCullMode(
        const FMaterial& InMaterialResource,
        const FMeshBatch& InMeshBatch,
        bool bReverseCulling = false);
    static FMeshDrawingPolicyOverrideSettings ComputeMeshOverrideSettings(
        const FMeshBatch& InMesh);
};
```

---

## FMeshPassProcessorRenderState

```cpp
struct FMeshPassProcessorRenderState
{
    // ─── セッター ───
    void SetBlendState(FRHIBlendState* InBlendState);
    void SetDepthStencilState(FRHIDepthStencilState* InDepthStencilState);
    void SetStencilRef(uint32 InStencilRef);
    void SetViewUniformBuffer(FRHIUniformBuffer* InViewUniformBuffer);
    void SetPassUniformBuffer(FRHIUniformBuffer* InPassUniformBuffer);
    void SetDepthStencilAccess(FExclusiveDepthStencil::Type InDepthStencilAccess);
    void SetNullDepthStencilState();

    // ─── ゲッター ───
    FRHIBlendState*              GetBlendState()          const;
    FRHIDepthStencilState*       GetDepthStencilState()   const;
    uint32                       GetStencilRef()          const;
    FRHIUniformBuffer*           GetViewUniformBuffer()   const;
    FRHIUniformBuffer*           GetPassUniformBuffer()   const;
    FExclusiveDepthStencil::Type GetDepthStencilAccess()  const;

    // ─── PSO への適用 ───
    void ApplyToPSO(FGraphicsPipelineStateInitializer& GraphicsPSOInit) const;
};
```

---

## EMeshPassFeatures

```cpp
enum class EMeshPassFeatures {
    Default             = 0,
    PositionOnly        = 1 << 0,
    PositionAndNormalOnly = 1 << 1,
};
```

---

## FMeshPassDrawListContext（書き込み先インターフェース）

```cpp
class FMeshPassDrawListContext
{
public:
    virtual FMeshDrawCommand& AddCommand(
        FMeshDrawCommand& Initializer, uint32 NumElements) = 0;
    virtual void FinalizeCommand(
        const FMeshBatch& MeshBatch, int32 BatchElementIndex,
        const FMeshDrawCommandPrimitiveIdInfo& IdInfo,
        ERasterizerFillMode MeshFillMode, ERasterizerCullMode MeshCullMode,
        FMeshDrawCommandSortKey SortKey,
        EFVisibleMeshDrawCommandFlags Flags,
        const FGraphicsMinimalPipelineStateInitializer& PipelineState,
        const FMeshProcessorShaders* ShadersForDebugging,
        FMeshDrawCommand& MeshDrawCommand) = 0;
};

// 静的コマンドキャッシュ書き込み先
class FCachedPassMeshDrawListContext : public FMeshPassDrawListContext { ... };

// 動的コマンドフレーム一時バッファ書き込み先
class FDynamicPassMeshDrawListContext : public FMeshPassDrawListContext { ... };
```

---

## パス登録マクロ

```cpp
// パス登録（ShadingPath・MeshPass・MeshPassFlags の紐付け）
REGISTER_MESHPASSPROCESSOR_AND_PSOCOLLECTOR(
    PassName,              // 識別名（ログ等に使用）
    FactoryFunction,       // FMeshPassProcessor* (*Factory)(...)
    EShadingPath,          // Deferred / Mobile / Forward
    EMeshPass::Type,
    EMeshPassFlags)

// MeshPassFlags の値
enum class EMeshPassFlags : uint8 {
    None                = 0,
    CachedMeshCommands  = 1 << 0,  // 静的コマンドキャッシュを使用
    MainView            = 1 << 1,  // メインビューパス
    Shadows             = 1 << 2,  // シャドウパス
};
```

---

## SubmitMeshDrawCommands

```cpp
// コマンド配列を RHI コマンドリストに発行
void SubmitMeshDrawCommands(
    const FMeshCommandOneFrameArray& VisibleMeshDrawCommands,
    const FGraphicsMinimalPipelineStateSet& GraphicsMinimalPipelineStateSet,
    FRHIBuffer* SceneArgs,
    const void* ViewIdentity,
    bool bDynamicInstancing,
    uint32 InstanceFactor,
    FRHICommandList& RHICmdList);

// FParallelCommandListSet を使った並列発行
void SubmitMeshDrawCommandsRange(
    const FMeshCommandOneFrameArray& VisibleMeshDrawCommands,
    const FGraphicsMinimalPipelineStateSet& GraphicsMinimalPipelineStateSet,
    FRHIBuffer* SceneArgs,
    const void* ViewIdentity,
    bool bDynamicInstancing,
    int32 StartIndex,
    int32 NumDraws,
    uint32 InstanceFactor,
    FRHICommandList& RHICmdList);
```
