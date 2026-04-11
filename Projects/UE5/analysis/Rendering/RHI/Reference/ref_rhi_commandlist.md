# REF: FRHICommandList 系

- 対象ファイル: `Runtime/RHI/Public/RHICommandList.h`
- 関連Details: [[a_rhi_abstraction]]

---

## クラス継承

```
FRHICommandListBase          ← メモリスタック・コマンドリンクリスト管理
  └── FRHICommandList        ← 遅延実行コマンドリスト（並列記録可）
        └── FRHIComputeCommandList ← Compute 系コマンドのみ
              └── FRHICommandListImmediate ← 即時実行型（GRHICommandList）
```

---

## FRHICommandListBase

```cpp
class FRHICommandListBase
{
public:
    // コマンドメモリアロケーション（FMemStack ベース）
    void* Alloc(int32 AllocSize, int32 Alignment);

    // RHI スレッドを待機
    void WaitForRHIThreadTasks();
    void WaitForDispatch();

    // コマンドリストが空か
    bool IsImmediate() const;
    bool IsImmediateAsyncCompute() const;

    // フラッシュ（RHI スレッドに投入して完了を待つ）
    void ImmediateFlush(EImmediateFlushType::Type FlushType);

    // GPUマスク（マルチGPU）
    FRHIGPUMask GetGPUMask() const;
};
```

---

## FRHICommandList（主要コマンド）

```cpp
class FRHICommandList : public FRHICommandListBase
{
public:
    // ---- レンダーパス ----
    void BeginRenderPass(
        const FRHIRenderPassInfo& InInfo,
        const TCHAR* InName);
    void EndRenderPass();

    // ---- パイプラインステート ----
    void SetGraphicsPipelineState(
        FRHIGraphicsPipelineState* GraphicsState,
        uint32 StencilRef,
        bool bApplyAdditionalState = false);
    void SetComputePipelineState(
        FRHIComputePipelineState* ComputePipelineState,
        FRHIComputeShader* ComputeShader);

    // ---- ビューポート / シザー ----
    void SetViewport(float MinX, float MinY, float MinZ,
                     float MaxX, float MaxY, float MaxZ);
    void SetScissorRect(bool bEnable,
                        uint32 MinX, uint32 MinY,
                        uint32 MaxX, uint32 MaxY);

    // ---- 頂点 / インデックス ----
    void SetStreamSource(
        uint32 StreamIndex, FRHIBuffer* VertexBuffer, uint32 Offset);

    // ---- シェーダーリソースバインド ----
    void SetShaderTexture(
        FRHIGraphicsShader* Shader,
        uint32 TextureIndex,
        FRHITexture* NewTexture);
    void SetShaderResourceViewParameter(
        FRHIGraphicsShader* Shader,
        uint32 SamplerIndex,
        FRHIShaderResourceView* SRV);
    void SetShaderUnorderedAccessViewParameter(
        FRHIComputeShader* PixelShader,
        uint32 UAVIndex,
        FRHIUnorderedAccessView* UAV);
    void SetShaderSampler(
        FRHIGraphicsShader* Shader,
        uint32 SamplerIndex,
        FRHISamplerState* NewState);
    void SetUniformBufferParameter(
        FRHIGraphicsShader* Shader,
        const FShaderUniformBufferParameter& Parameter,
        FRHIUniformBuffer* UniformBuffer);

    // ---- ドローコール ----
    void DrawPrimitive(
        uint32 BaseVertexIndex, uint32 NumPrimitives, uint32 NumInstances);
    void DrawIndexedPrimitive(
        FRHIBuffer* IndexBuffer,
        int32 BaseVertexIndex,
        uint32 FirstInstance,
        uint32 NumVertices,
        uint32 StartIndex,
        uint32 NumPrimitives,
        uint32 NumInstances);
    void DrawIndirect(
        FRHIBuffer* ArgumentBuffer, uint32 ArgumentOffset);
    void DrawIndexedIndirect(
        FRHIBuffer* IndexBufferRHI,
        FRHIBuffer* ArgumentsBufferRHI,
        uint32 DrawArgumentsIndex,
        uint32 NumInstances);

    // ---- コンピュート ----
    void DispatchComputeShader(
        uint32 ThreadGroupCountX,
        uint32 ThreadGroupCountY,
        uint32 ThreadGroupCountZ);
    void DispatchIndirectComputeShader(
        FRHIBuffer* ArgumentBuffer, uint32 ArgumentOffset);

    // ---- コピー ----
    void CopyToResolveTarget(
        FRHITexture* SourceTexture,
        FRHITexture* DestTexture,
        const FResolveParams& ResolveParams);
    void CopyTexture(
        FRHITexture* SourceTexture,
        FRHITexture* DestTexture,
        const FRHICopyTextureInfo& CopyInfo);
    void UpdateBuffer(
        FRHIBuffer* Buffer,
        uint32 Offset,
        uint32 Size,
        const void* Data);

    // ---- バリア（遷移）----
    void Transition(TArrayView<const FRHITransitionInfo> Infos);

    // ---- クエリ ----
    void BeginOcclusionQueryBatch(uint32 NumQueriesInBatch);
    void EndOcclusionQueryBatch();
    void BeginRenderQuery(FRHIRenderQuery* RenderQuery);
    void EndRenderQuery(FRHIRenderQuery* RenderQuery);
};
```

---

## FRHICommandListImmediate

```cpp
class FRHICommandListImmediate : public FRHICommandList
{
public:
    // 同期実行（コマンドを即座に RHI スレッドに送り完了を待つ）
    void ImmediateFlush(EImmediateFlushType::Type FlushType);
    // FlushType:
    //   WaitForOutstandingTasksOnly … タスク完了待ちのみ
    //   DispatchToRHIThread        … RHI スレッドにディスパッチ
    //   WaitForDispatchToRHIThread … ディスパッチ + 受け取り確認待ち
    //   FlushRHIThread             … RHI スレッドの全コマンド完了待ち
    //   FlushRHIThreadFlushResources… リソースも一緒にフラッシュ

    // GPU フェンス待機
    void WaitForFence(FRHIComputeFence* Fence);

    // GPU 完了待ち（フレームスタール）
    void BlockUntilGPUIdle();
};
```

### グローバルアクセス

```cpp
extern RHI_API FRHICommandListImmediate GRHICommandList;
// アクセス関数
inline FRHICommandListImmediate& FRHICommandListExecutor::GetImmediateCommandList()
{
    return GRHICommandList;
}
```

---

## ERHIThreadMode

```cpp
enum class ERHIThreadMode
{
    None,             // RHI スレッドなし（レンダースレッドが直接実行）
    DedicatedThread,  // 専用 RHI スレッド（デフォルト）
    Tasks,            // タスクグラフ上で実行
};
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RHIThread.Enable` | 1 | RHI スレッド有効（0=無効） |
| `r.RHIThread.EnableTaskMode` | 0 | タスクモード有効 |
| `r.RHI.MaximumFrameLatency` | 2 | 最大フレームレイテンシ（GPU 先行フレーム数）|

---

> [!note]- FRHICommandList のコマンドキュー構造とメモリスタック
> `FRHICommandListBase` はコマンドを `FMemStack`（スレッドローカルメモリスタック）上にアロケートするため malloc を呼ばず非常に高速。  
> 各コマンドは `FRHICommandBase` を継承した構造体として積まれ、リンクリストで繋がれる。  
> `FRHICommandListBase::Execute()` (`RHICommandList.cpp:518`) がリストを先頭から順番にたどり `IRHICommandContext` の仮想関数を呼ぶ仕組みになっている。

> [!note]- ImmediateFlush の各モードと CPU/GPU スタール
> `ImmediateFlush()` (`RHICommandList.cpp:1573`) の `FlushType` は 5 段階あり、`DispatchToRHIThread` はコマンドをキューに投げて即座に戻るが、`FlushRHIThread` は RHI スレッドの全コマンドが完了するまで CPU がスタールする。  
> `BlockUntilGPUIdle()` はさらに GPU フェンスの完了まで待つ。`FlushRenderingCommands()` はゲームスレッドからレンダースレッドを同期させる高レベル API であり、これらは重いため頻繁な呼び出しは避けるべき。

> [!note]- ERHIThreadMode と r.RHIThread.Enable
> `r.RHIThread.Enable=1`（デフォルト）で専用 RHI スレッドが有効になり、`ERHIThreadMode::DedicatedThread` として動作する。  
> `r.RHIThread.Enable=0` にするとレンダースレッドが直接 `IRHICommandContext` を呼ぶ `None` モードになり、スレッド間オーバーヘッドがなくなるが CPU コアの分離が失われる。  
> `r.RHIThread.EnableTaskMode=1` は実験的な Task グラフモード（`ERHIThreadMode::Tasks`）で、RHI コマンドを複数スレッドで並列処理できるが対応プラットフォームが限定される。
