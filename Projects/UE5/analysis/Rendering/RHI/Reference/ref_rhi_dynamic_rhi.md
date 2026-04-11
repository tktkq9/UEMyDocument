# REF: FDynamicRHI / IRHICommandContext

- 対象ファイル: `Runtime/RHI/Public/DynamicRHI.h`, `RHIContext.h`
- 関連Details: [[a_rhi_abstraction]]

---

## FDynamicRHI（リソース生成・破棄インターフェース）

```cpp
class FDynamicRHI
{
public:
    // ---- 初期化 / 終了 ----
    virtual void Init()                                   = 0;
    virtual void PostInit()                               {}
    virtual void Shutdown()                               = 0;
    virtual const TCHAR* GetName()                        = 0;  // "D3D12" / "Vulkan" 等

    // ---- テクスチャ ----
    virtual FTextureRHIRef RHICreateTexture(
        const FRHITextureCreateDesc& CreateDesc)          = 0;

    virtual void RHIUpdateTexture2D(
        FRHICommandListBase& RHICmdList,
        FRHITexture* Texture,
        uint32 MipIndex,
        const struct FUpdateTextureRegion2D& UpdateRegion,
        uint32 SourcePitch,
        const uint8* SourceData)                          = 0;

    // ---- バッファ ----
    virtual FBufferRHIRef RHICreateBuffer(
        FRHICommandListBase& RHICmdList,
        const FRHIBufferCreateDesc& CreateDesc,
        ERHIAccess InResourceState)                       = 0;

    virtual void* RHILockBuffer(
        FRHICommandListBase& RHICmdList,
        FRHIBuffer* Buffer,
        uint32 Offset, uint32 Size,
        EResourceLockMode LockMode)                       = 0;

    virtual void RHIUnlockBuffer(
        FRHICommandListBase& RHICmdList,
        FRHIBuffer* Buffer)                               = 0;

    // ---- ユニフォームバッファ ----
    virtual FUniformBufferRHIRef RHICreateUniformBuffer(
        const void* Contents,
        const FRHIUniformBufferLayout* Layout,
        EUniformBufferUsage Usage,
        EUniformBufferValidation Validation = EUniformBufferValidation::ValidateResources) = 0;

    // ---- シェーダー ----
    virtual FVertexShaderRHIRef    RHICreateVertexShader(
        TArrayView<const uint8> Code, const FSHAHash& Hash)   = 0;
    virtual FPixelShaderRHIRef     RHICreatePixelShader(
        TArrayView<const uint8> Code, const FSHAHash& Hash)   = 0;
    virtual FComputeShaderRHIRef   RHICreateComputeShader(
        TArrayView<const uint8> Code, const FSHAHash& Hash)   = 0;
    virtual FMeshShaderRHIRef      RHICreateMeshShader(
        TArrayView<const uint8> Code, const FSHAHash& Hash)   = 0;

    // ---- PSO ----
    virtual FGraphicsPipelineStateRHIRef RHICreateGraphicsPipelineState(
        const FGraphicsPipelineStateInitializer& Initializer) = 0;
    virtual FComputePipelineStateRHIRef  RHICreateComputePipelineState(
        const FRHIComputeShader* ComputeShader)               = 0;

    // ---- ビュー ----
    virtual FShaderResourceViewRHIRef  RHICreateShaderResourceView(
        FRHITexture* Texture,
        const FRHIViewDesc& ViewDesc)                         = 0;
    virtual FShaderResourceViewRHIRef  RHICreateShaderResourceView(
        FRHIBuffer* Buffer,
        const FRHIViewDesc& ViewDesc)                         = 0;
    virtual FUnorderedAccessViewRHIRef RHICreateUnorderedAccessView(
        FRHITexture* Texture,
        uint32 MipLevel, uint16 FirstArraySlice, uint16 NumArraySlices) = 0;
    virtual FUnorderedAccessViewRHIRef RHICreateUnorderedAccessView(
        FRHIBuffer* Buffer,
        bool bUseUAVCounter, bool bAppendBuffer)              = 0;

    // ---- ステート ----
    virtual FBlendStateRHIRef        RHICreateBlendState(
        const FBlendStateInitializerRHI& Initializer)         = 0;
    virtual FRasterizerStateRHIRef   RHICreateRasterizerState(
        const FRasterizerStateInitializerRHI& Initializer)    = 0;
    virtual FDepthStencilStateRHIRef RHICreateDepthStencilState(
        const FDepthStencilStateInitializerRHI& Initializer)  = 0;
    virtual FSamplerStateRHIRef      RHICreateSamplerState(
        const FSamplerStateInitializerRHI& Initializer)       = 0;

    // ---- フレーム制御 ----
    virtual void RHIBeginFrame()                              = 0;
    virtual void RHIEndFrame(const FRHIEndFrameArgs& Args)    = 0;
    virtual void RHIBeginScene()                              {}
    virtual void RHIEndScene()                                {}

    // ---- GPU 情報 ----
    virtual void RHIGetTextureMemoryStats(FTextureMemoryStats& OutStats) = 0;
    virtual bool RHIGetAvailableResolutions(
        FScreenResolutionArray& Resolutions, bool bIgnoreRefreshRate)     = 0;
};

// グローバルポインタ（プロセスで 1 つ）
extern RHI_API FDynamicRHI* GDynamicRHI;
```

---

## IRHICommandContext（コマンド記録インターフェース）

```cpp
class IRHICommandContext : public IRHIComputeContext
{
public:
    // ---- パイプライン ----
    virtual void RHISetGraphicsPipelineState(
        FRHIGraphicsPipelineState* GraphicsState,
        uint32 StencilRef,
        bool bApplyAdditionalState)                       = 0;

    // ---- レンダーターゲット ----
    virtual void RHISetRenderTargets(
        uint32 NumSimultaneousRenderTargets,
        const FRHIRenderTargetView* NewRenderTargetsRHI,
        const FRHIDepthRenderTargetView* NewDepthStencilTargetRHI) = 0;
    virtual void RHIBeginRenderPass(
        const FRHIRenderPassInfo& InInfo,
        const TCHAR* InName)                              = 0;
    virtual void RHIEndRenderPass()                       = 0;

    // ---- ジオメトリ ----
    virtual void RHISetStreamSource(
        uint32 StreamIndex,
        FRHIBuffer* VertexBuffer,
        uint32 Offset)                                    = 0;
    virtual void RHISetViewport(float MinX, float MinY, float MinZ,
                                float MaxX, float MaxY, float MaxZ) = 0;
    virtual void RHISetScissorRect(bool bEnable,
                                   uint32 MinX, uint32 MinY,
                                   uint32 MaxX, uint32 MaxY) = 0;

    // ---- ドローコール ----
    virtual void RHIDrawPrimitive(
        uint32 BaseVertexIndex,
        uint32 NumPrimitives,
        uint32 NumInstances)                              = 0;
    virtual void RHIDrawIndexedPrimitive(
        FRHIBuffer* IndexBuffer,
        int32 BaseVertexIndex,
        uint32 FirstInstance,
        uint32 NumVertices,
        uint32 StartIndex,
        uint32 NumPrimitives,
        uint32 NumInstances)                              = 0;

    // ---- シェーダーリソース ----
    virtual void RHISetShaderTexture(
        FRHIGraphicsShader* Shader,
        uint32 TextureIndex,
        FRHITexture* NewTexture)                          = 0;
    virtual void RHISetShaderSampler(
        FRHIGraphicsShader* Shader,
        uint32 SamplerIndex,
        FRHISamplerState* NewState)                       = 0;
    virtual void RHISetShaderUniformBuffer(
        FRHIGraphicsShader* Shader,
        uint32 BufferIndex,
        FRHIUniformBuffer* Buffer)                        = 0;
};

// IRHIComputeContext（Compute 系）
class IRHIComputeContext
{
public:
    virtual void RHISetComputePipelineState(
        FRHIComputePipelineState* ComputePipelineState)   = 0;
    virtual void RHIDispatchComputeShader(
        uint32 ThreadGroupCountX, uint32 ThreadGroupCountY,
        uint32 ThreadGroupCountZ)                         = 0;
    virtual void RHIDispatchIndirectComputeShader(
        FRHIBuffer* ArgumentBuffer, uint32 ArgumentOffset) = 0;

    // リソーストランジション
    virtual void RHISetTrackedAccess(
        const FRHICommandSetTrackedAccess& Args)          = 0;
};
```

---

## グローバル RHI 関数（ショートカット）

```cpp
// GDynamicRHI の仮想関数を直接呼ぶショートカット
// （実装は RHI.h 内のインライン関数）

inline FTextureRHIRef RHICreateTexture(const FRHITextureCreateDesc& CreateDesc);
inline FBufferRHIRef  RHICreateBuffer(FRHICommandListBase&, const FRHIBufferCreateDesc&, ERHIAccess);
inline FUniformBufferRHIRef RHICreateUniformBuffer(const void*, const FRHIUniformBufferLayout*, EUniformBufferUsage);
inline FVertexShaderRHIRef  RHICreateVertexShader(TArrayView<const uint8>, const FSHAHash&);
inline FPixelShaderRHIRef   RHICreatePixelShader(TArrayView<const uint8>, const FSHAHash&);
inline FComputeShaderRHIRef RHICreateComputeShader(TArrayView<const uint8>, const FSHAHash&);
inline FBlendStateRHIRef    RHICreateBlendState(const FBlendStateInitializerRHI&);
inline FRasterizerStateRHIRef RHICreateRasterizerState(const FRasterizerStateInitializerRHI&);
inline FDepthStencilStateRHIRef RHICreateDepthStencilState(const FDepthStencilStateInitializerRHI&);
inline FSamplerStateRHIRef  RHICreateSamplerState(const FSamplerStateInitializerRHI&);

// 機能確認
inline ERHIFeatureLevel::Type RHIGetFeatureLevel();
inline bool RHISupportsRayTracing(EShaderPlatform Platform);
inline bool RHISupportsComputeShaders(EShaderPlatform Platform);
inline bool RHISupportsBindless(EShaderPlatform Platform);
```

---

## ERHIFeatureLevel

```cpp
namespace ERHIFeatureLevel
{
    enum Type
    {
        ES2_REMOVED,   // 削除済み
        ES3_1,         // OpenGL ES 3.1 / Metal iOS / Vulkan mobile
        SM4_REMOVED,   // 削除済み
        SM5,           // DX11 SM5 (D3D12 / Vulkan)
        SM6,           // DX12 SM6 (Mesh Shader / DXR)
        Num
    };
}
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RHI.RayTracing` | 1 | レイトレーシング対応（SM6 以上）|
| `r.RHI.MeshShaders` | 1 | Mesh / Amplification Shader 有効 |
| `r.RHI.Bindless` | 0 | バインドレスリソース有効 |
| `r.D3D12.MultiGPU` | 0 | マルチ GPU（SLI / mGPU）有効 |

---

> [!note]- FDynamicRHI のプラグインロードと GDynamicRHI 初期化
> `FDynamicRHI` の実装は実行時にプラグインとしてロードされる。起動時に `FModuleManager::Get().LoadModule("D3D12RHI")` が呼ばれ、モジュールの `StartupModule()` 内で `GDynamicRHI = new FD3D12DynamicRHI(...)` が設定される。  
> グローバルポインタ `GDynamicRHI` は以降すべての `RHICreateXxx()` インライン関数から参照される。プラットフォーム判定は起動引数や `r.GraphicsAPI` CVar で行われる。

> [!note]- IRHICommandContext と FDynamicRHI の役割分担
> `FDynamicRHI` はリソース**生成・破棄**の責務を持ち、`IRHICommandContext` はコマンド**記録**の責務を持つ。  
> これらは意図的に分離されており、D3D12 実装では `FD3D12DynamicRHI` が `FDynamicRHI` を継承し、`FD3D12CommandContext` が `IRHICommandContext` を継承する（別クラス）。  
> コマンドリストは複数存在できる（並列記録対応）が、リソース生成は常に `GDynamicRHI` の1インスタンスを通す。

> [!note]- ERHIFeatureLevel::SM6 と D3D12 Shader Model 6 機能
> `SM6` フィーチャーレベルは DX12 の Shader Model 6.x を前提とし、Mesh Shader / Amplification Shader / DXR（レイトレーシング）/ Bindless が有効になる。  
> `RHISupportsRayTracing(Platform)` は `SM6` かつ `r.RHI.RayTracing=1` の場合に `true` を返す。  
> コンソール向けプラットフォームでは独自の機能レベルが定義されており、`SM6` に対応するが一部機能の可用性は異なる。
