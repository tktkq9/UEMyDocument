# REF: D3D12 コンテキスト系クラス

- 対象ファイル: `D3D12RHI/Private/D3D12CommandContext.h/.cpp`, `D3D12CommandList.h/.cpp`
- 関連Details: [[d_rhi_d3d12_context]]

---

## FD3D12ContextCommon（主クラス）

```cpp
class FD3D12ContextCommon : public IRHICommandContext
{
public:
    // ---- コマンドリスト管理 ----
    void OpenCommandList(
        FD3D12CommandAllocator* Allocator = nullptr,
        ID3D12PipelineState* InitialPSO = nullptr);
    void CloseCommandList();

    // バリアの保留リストをフラッシュ（ドローコール直前等）
    void FlushResourceBarriers();

    // 保留バリアを追加（RDG から呼ばれる）
    void AddPendingResourceBarrier(
        FD3D12Resource* Resource,
        D3D12_RESOURCE_STATES Before,
        D3D12_RESOURCE_STATES After);

    // ---- IRHICommandContext 仮想関数（主要なもの）----

    // パイプライン
    virtual void RHISetGraphicsPipelineState(
        FRHIGraphicsPipelineState* GraphicsState,
        uint32 StencilRef, bool bApplyAdditionalState) override;
    virtual void RHISetComputePipelineState(
        FRHIComputePipelineState* ComputePipelineState) override;

    // レンダーターゲット
    virtual void RHIBeginRenderPass(
        const FRHIRenderPassInfo& InInfo, const TCHAR* InName) override;
    virtual void RHIEndRenderPass() override;
    virtual void RHISetRenderTargets(
        uint32 NumSimultaneousRenderTargets,
        const FRHIRenderTargetView* NewRenderTargetsRHI,
        const FRHIDepthRenderTargetView* NewDepthStencilTargetRHI) override;

    // ジオメトリ
    virtual void RHISetStreamSource(
        uint32 StreamIndex, FRHIBuffer* VertexBuffer, uint32 Offset) override;
    virtual void RHISetViewport(
        float MinX, float MinY, float MinZ,
        float MaxX, float MaxY, float MaxZ) override;
    virtual void RHISetScissorRect(
        bool bEnable, uint32 MinX, uint32 MinY, uint32 MaxX, uint32 MaxY) override;

    // ドローコール
    virtual void RHIDrawPrimitive(
        uint32 BaseVertexIndex, uint32 NumPrimitives, uint32 NumInstances) override;
    virtual void RHIDrawIndexedPrimitive(
        FRHIBuffer* IndexBuffer,
        int32 BaseVertexIndex, uint32 FirstInstance, uint32 NumVertices,
        uint32 StartIndex, uint32 NumPrimitives, uint32 NumInstances) override;
    virtual void RHIDrawIndirect(
        FRHIBuffer* ArgumentBuffer, uint32 ArgumentOffset) override;

    // コンピュート
    virtual void RHIDispatchComputeShader(
        uint32 X, uint32 Y, uint32 Z) override;
    virtual void RHIDispatchIndirectComputeShader(
        FRHIBuffer* ArgumentBuffer, uint32 ArgumentOffset) override;

    // コピー
    virtual void RHICopyTexture(
        FRHITexture* SourceTexture, FRHITexture* DestTexture,
        const FRHICopyTextureInfo& CopyInfo) override;
    virtual void RHICopyBufferRegion(
        FRHIBuffer* DestBuffer, uint64 DstOffset,
        FRHIBuffer* SourceBuffer, uint64 SrcOffset, uint64 NumBytes) override;

    // バリア
    virtual void RHISetTrackedAccess(
        const FRHICommandSetTrackedAccess& Args) override;

    // クエリ
    virtual void RHIBeginOcclusionQueryBatch(uint32 NumQueriesInBatch) override;
    virtual void RHIEndOcclusionQueryBatch() override;
    virtual void RHIBeginRenderQuery(FRHIRenderQuery* RenderQuery) override;
    virtual void RHIEndRenderQuery(FRHIRenderQuery* RenderQuery) override;

protected:
    FD3D12CommandList*        CommandListHandle;   // 現在記録中のコマンドリスト
    FD3D12Device*             ParentDevice;
    FD3D12Queue*              Queue;
    FD3D12StateCachePrivate   StateCache;          // 冗長設定フィルタリング
    FD3D12ExplicitDescriptorCache DescriptorCache;// ディスクリプタコピーキャッシュ

    // 保留中バリアバッファ
    TArray<D3D12_RESOURCE_BARRIER, TInlineAllocator<16>> PendingBarriers;
};
```

---

## FD3D12DeferredDeleteObject

```cpp
struct FD3D12DeferredDeleteObject
{
    enum class EType
    {
        RHIObject,            // FD3D12Resource 等
        D3DObject,            // ID3D12Object 系
        Heap,                 // FD3D12Heap
        DescriptorHeap,       // FD3D12DescriptorHeap
        BindlessDescriptor,   // バインドレスディスクリプタ（PLATFORM_SUPPORTS_BINDLESS_RENDERING）
        CPUAllocation,        // CPU メモリ
        DescriptorBlock,      // ディスクリプタブロック
        VirtualAllocation,    // 仮想アドレスアロケーション
        Func,                 // 任意の TUniqueFunction<void()>
        TextureStagingBuffer, // テクスチャステージングバッファ
    } Type;

    // Type に応じたユニオンデータ
    union { FD3D12Resource* RHIObject; ID3D12Object* D3DObject; /* … */ };
};
```

---

## FD3D12CommandList

```cpp
class FD3D12CommandList
{
public:
    // 記録開始・終了
    void Open(
        FD3D12CommandAllocator* Allocator,
        ID3D12PipelineState* InitialPSO = nullptr);
    void Close();

    // D3D12 ネイティブ
    ID3D12GraphicsCommandList*  GetCommandList()  const;
    ID3D12GraphicsCommandList2* GetCommandList2() const;  // WriteBuf 系 API
    ID3D12GraphicsCommandList4* GetCommandList4() const;  // DXR
    ID3D12GraphicsCommandList6* GetCommandList6() const;  // Mesh Shader

    // フェンス・アロケータ
    uint64                  GetFenceValue()       const;
    FD3D12CommandAllocator* GetCurrentAllocator() const;

    // コマンドリストタイプ
    D3D12_COMMAND_LIST_TYPE GetType() const;
    // D3D12_COMMAND_LIST_TYPE_DIRECT / COMPUTE / COPY
};
```

---

## FD3D12CommandAllocator

```cpp
class FD3D12CommandAllocator
{
public:
    // GPU 完了後に Reset して再利用
    void Reset();

    // このアロケータのコマンドが GPU で完了したか（FenceValue 確認）
    bool IsReady() const;

    ID3D12CommandAllocator* GetCommandAllocator() const;

    // 対応するキュータイプ
    ED3D12QueueType GetQueueType() const;
};
```

### アロケータ再利用プール

```
FD3D12CommandAllocatorManager
  ├── TQueue<FD3D12CommandAllocator*> FreeAllocators  ← 再利用可能なもの
  └── TArray<FD3D12CommandAllocator*> AllAllocators   ← 生成済み全て

[フレーム記録]
1. FreeAllocators から取り出して Open()
2. コマンド記録
3. Close() → ExecuteCommandLists()
4. Signal() でフェンス値を記録
5. GetLastCompletedFence() >= フェンス値 → Reset() → FreeAllocators に返す
```

---

## ED3D12FlushFlags

```cpp
enum class ED3D12FlushFlags : uint8
{
    None           = 0,
    WaitForCompletion = 1 << 0,  // GPU 完了まで CPU ブロック
    WaitForPipeline   = 1 << 1,  // パイプライン完了待ち
    FlushBeforeSubmit = 1 << 2,  // 投入前にフラッシュ
};
```

---

## FD3D12StateCachePrivate（ステートキャッシュ）

```cpp
class FD3D12StateCachePrivate
{
public:
    // キャッシュに変化があれば D3D12 API を呼ぶ
    void ApplyState(
        ED3D12PipelineType PipelineType,
        FD3D12ContextCommon& Context);

    // キャッシュのクリア（コマンドリスト切り替え時）
    void ClearState();

private:
    // キャッシュデータ
    FD3D12PipelineState*     CurrentGraphicsPipelineState;
    FD3D12PipelineState*     CurrentComputePipelineState;
    D3D12_VIEWPORT           Viewport;
    D3D12_RECT               ScissorRect;
    FRHIDepthStencilState*   DepthStencilState;
    uint32                   StencilRef;
    FRHIBlendState*          BlendState;
    FLinearColor             BlendFactor;
    FD3D12RootSignature*     CurrentRootSignature;
};
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.D3D12.MaxCommandsPerCommandList` | 10000 | 1コマンドリストあたりの最大コマンド数 |
| `r.D3D12.BarrierBatchSize` | 16 | バリアバッチサイズ（PendingBarriers の最大数）|
| `r.D3D12.EnableAsyncCompute` | 1 | 非同期コンピュートキュー有効 |
| `r.D3D12.ForceAsyncCompute` | 0 | Compute を強制的に非同期キューへ |
