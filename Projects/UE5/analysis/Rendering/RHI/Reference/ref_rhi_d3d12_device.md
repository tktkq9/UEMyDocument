# REF: D3D12 デバイス系クラス

- 対象ファイル: `D3D12RHI/Private/D3D12Device.h/.cpp`, `D3D12Queue.h`, `D3D12Descriptors.h`, `D3D12Submission.h`
- 関連Details: [[c_rhi_d3d12_device]]

---

## FD3D12DynamicRHI

```cpp
class FD3D12DynamicRHI : public FDynamicRHI
{
public:
    // ---- 初期化 ----
    virtual void Init()     override;  // アダプタ列挙・デバイス生成
    virtual void Shutdown() override;

    // ---- アダプタ管理 ----
    uint32           GetNumAdapters()          const;
    FD3D12Adapter*   GetAdapter(uint32 Index = 0);

    // ---- リソース生成（FDynamicRHI 実装）----
    virtual FTextureRHIRef RHICreateTexture(
        const FRHITextureCreateDesc& CreateDesc) override;
    virtual FBufferRHIRef  RHICreateBuffer(
        FRHICommandListBase& RHICmdList,
        const FRHIBufferCreateDesc& CreateDesc,
        ERHIAccess InResourceState) override;

    // ---- Present ----
    virtual void RHIEndDrawingViewport(
        FRHIViewport* Viewport, bool bPresent, bool bLockToVsync) override;

    // ---- ネイティブアクセス（ID3D12DynamicRHI 経由）----
    ID3D12Device*         RHIGetNativeDevice();
    ID3D12CommandQueue*   RHIGetNativeCommandQueue();
    IDXGIFactory2*        RHIGetNativeDXGIFactory2();
};
```

---

## FD3D12Adapter

```cpp
class FD3D12Adapter
{
public:
    // デバイスアクセス
    FD3D12Device* GetDevice(uint32 GPUIndex = 0);
    uint32        GetNumDevices() const;

    // DXGI / D3D12 ネイティブ
    IDXGIFactory4*     GetDXGIFactory4()  const;
    IDXGIAdapter*      GetAdapter()       const;
    D3D_FEATURE_LEVEL  GetFeatureLevel()  const;
    D3D_SHADER_MODEL   GetHighestShaderModel() const;  // SM6.5 等

    // キャッシュ（アダプタ内で共有）
    FD3D12RootSignatureManager  RootSignatureManager;
    FD3D12PipelineStateCache    PipelineStateCache;    // ディスクキャッシュ対応

    // レイトレーシングパイプラインキャッシュ
    FD3D12RayTracingPipelineCache* RayTracingPipelineCache;
};
```

---

## FD3D12Device

```cpp
class FD3D12Device
{
public:
    // ---- D3D12 ネイティブ ----
    ID3D12Device*  GetDevice()  const;
    ID3D12Device5* GetDevice5() const;  // DXR (RTPSO 作成等)
    ID3D12Device6* GetDevice6() const;  // SetBackgroundProcessingMode

    // ---- キュー ----
    FD3D12Queue& GetQueue(ED3D12QueueType QueueType);
    // QueueType: Direct / AsyncCompute / Copy

    // ---- ディスクリプタ管理 ----
    FD3D12DescriptorManager& GetDescriptorManager();
    FD3D12BindlessDescriptors* GetBindlessDescriptors();  // nullptr=未対応

    // ---- GPU マスク（マルチ GPU）----
    FRHIGPUMask GetGPUMask() const;

    // ---- 遅延削除キュー ----
    void AddToDeferredDeleteQueue(FD3D12DeferredDeleteObject&& Object);
    void ProcessDeferredDeletionQueue(uint64 CompletedFenceValue);

    // ---- デバイス診断 ----
    FD3D12DiagnosticBuffer& GetDiagnosticBuffer();  // DRED 対応
    FD3D12Timing             Timing;                // GPU タイミング

    // ---- アロケーション ----
    FD3D12DefaultBufferAllocator DefaultBufferAllocator;
    FD3D12TextureAllocator        TextureAllocator;
};
```

---

## FD3D12Queue

```cpp
class FD3D12Queue
{
public:
    // D3D12 ネイティブ
    ID3D12CommandQueue* GetCommandQueue() const;

    // キュータイプ
    ED3D12QueueType GetQueueType() const;

    // フェンス操作
    uint64 Signal();                      // 現在値でシグナル → フェンス値を返す
    void   WaitForFence(uint64 Value);    // CPU ブロッキング待機
    uint64 GetLastCompletedFence() const; // GPU が完了した最新フェンス値
    uint64 GetCurrentFence()      const; // 次に Signal する予定の値

    // コマンド投入
    void ExecuteCommandList(FD3D12CommandList* CmdList);

    // キュー間同期（別 Queue の作業完了を待つ）
    void WaitForOtherQueue(FD3D12Queue& OtherQueue);
    void SignalForOtherQueue(FD3D12Queue& OtherQueue);

    // 診断バッファ（GPU クラッシュ検出）
    FD3D12DiagnosticBuffer DiagnosticBuffer;
};

enum class ED3D12QueueType : uint8
{
    Direct       = 0,  // D3D12_COMMAND_LIST_TYPE_DIRECT
    AsyncCompute = 1,  // D3D12_COMMAND_LIST_TYPE_COMPUTE
    Copy         = 2,  // D3D12_COMMAND_LIST_TYPE_COPY
};
```

---

## FD3D12Submission（投入スレッド管理）

```cpp
// 1フレーム分の投入ペイロード
struct FD3D12Payload
{
    TArray<FD3D12CommandList*> CommandLists;
    TArray<FD3D12FenceEventPair> PreWaitFences;    // 実行前に待つフェンス群
    TArray<FD3D12FenceEventPair> PostSignalFences; // 実行後にシグナルするフェンス群
    bool bFinalPresentPayload;                      // Present 直前フレームか
};
```

---

## FD3D12DescriptorManager（ディスクリプタヒープ）

```cpp
class FD3D12DescriptorManager
{
public:
    // CBV/SRV/UAV と Sampler の2種類のシェーダー可視ヒープを管理
    FD3D12DescriptorHeap& GetCurrentOnlineViewHeap();
    FD3D12DescriptorHeap& GetCurrentOnlineSamplerHeap();

    // オフラインヒープ（CPU 側、コピー元）
    FD3D12OfflineDescriptorManager& GetViewOfflineHeap(D3D12_DESCRIPTOR_HEAP_TYPE);

    // コマンドリストにヒープをバインド
    void SetDescriptorHeaps(ID3D12GraphicsCommandList* CmdList);
};
```

---

## FD3D12DiagnosticBuffer（DRED）

```cpp
class FD3D12DiagnosticBuffer : public FRHIDiagnosticBuffer
{
    // GPU クラッシュ後も参照可能な仮想ヒープ上のバッファ
    D3D12_GPU_VIRTUAL_ADDRESS GetGPUQueueData()      const;
    D3D12_GPU_VIRTUAL_ADDRESS GetGPUQueueMarkerIn()  const;  // ブレッドクラム IN
    D3D12_GPU_VIRTUAL_ADDRESS GetGPUQueueMarkerOut() const;  // ブレッドクラム OUT
    uint32 ReadMarkerIn()  const;  // CPU からクラッシュ位置読み取り
    uint32 ReadMarkerOut() const;
};
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.D3D12.UseD3DDebugDevice` | 0 | D3D12 デバッグレイヤー有効 |
| `r.D3D12.EnableDRED` | 0 | GPU クラッシュ診断（DRED）有効 |
| `r.D3D12.GPUTimeout` | 1 | GPU タイムアウト検出有効 |
| `r.D3D12.MaxCommandsPerCommandList` | 10000 | コマンドリスト分割閾値 |
| `r.D3D12.DescriptorHeapSize` | 500000 | グローバルビューヒープサイズ |
| `r.D3D12.BindlessResources` | 0 | バインドレスリソース有効 |

---

> [!note]- FD3D12Adapter と マルチ GPU（mGPU）構成
> `FD3D12Adapter` は `IDXGIAdapter` に対応し、複数 GPU を搭載したシステムでは `Adapter[0]`, `Adapter[1]` のように列挙される。  
> `r.D3D12.MultiGPU=1` を有効にすると1つの `FD3D12Adapter` が複数の `FD3D12Device` を持ち、`FRHIGPUMask` でどの GPU にコマンドを投入するかを制御できる。  
> デフォルトは `r.D3D12.MultiGPU=0` で、ゲーム向けにはほぼ使用しない（VR 等の特殊用途向け）。

> [!note]- FD3D12Queue の3種類と非同期コンピュート
> D3D12 では `Direct`（Graphics+Compute+Copy）・`AsyncCompute`（Compute+Copy）・`Copy` の3タイプのキューが使える。  
> `r.D3D12.EnableAsyncCompute=1`（デフォルト）で AsyncCompute キューが有効になり、Lumen の Radiance Cache 更新やシャドウマップ生成などが Graphics パスと並列実行される。  
> `FD3D12Queue::WaitForOtherQueue()` / `SignalForOtherQueue()` でキュー間のフェンス同期を行い、バリアなしで安全にリソースを共有できる。

> [!note]- FD3D12DiagnosticBuffer と DRED による GPU クラッシュ診断
> `r.D3D12.EnableDRED=1` を有効にすると `FD3D12DiagnosticBuffer` がページフォルト不可の仮想ヒープ上に確保される。  
> GPU クラッシュ（TDR）後もこのバッファは CPU から読み取り可能で、`ReadMarkerIn()`・`ReadMarkerOut()` でクラッシュ直前に実行していたブレッドクラムが分かる。  
> ブレッドクラムはコマンドリスト中に `RHI_BREADCRUMB_EVENT_F()` マクロで埋め込まれ、どのレンダリングパスで問題が起きたかを特定できる。
