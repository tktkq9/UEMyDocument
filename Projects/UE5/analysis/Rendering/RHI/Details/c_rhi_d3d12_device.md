# D3D12 デバイス層

- 対象: `Runtime/D3D12RHI/Private/D3D12Device.h/.cpp`, `D3D12Queue.h`, `D3D12Descriptors.h`
- 関連Reference: [[ref_rhi_d3d12_device]]

---

## 概要

D3D12 RHI のデバイス層は **FD3D12DynamicRHI → FD3D12Adapter → FD3D12Device → FD3D12Queue** の  
階層構造で GPU ハードウェアとの接続を管理する。  
`FD3D12DynamicRHI` がモジュール起動時に初期化され、アダプタ列挙・デバイス生成・キュー作成を行う。

---

## クラス階層

```
FD3D12DynamicRHI                    ← FDynamicRHI の D3D12 実装（プラグインエントリ）
  └── TArray<FD3D12Adapter*>        ← GPU アダプタ（マルチ GPU 対応）
        └── FD3D12Adapter           ← IDXGIAdapter ラッパー
              └── TArray<FD3D12Device*>  ← 論理デバイス（通常は1つ、SLI時は複数）
                    └── FD3D12Device← ID3D12Device ラッパー
                          ├── FD3D12Queue [Graphics] ← ID3D12CommandQueue (Graphics)
                          ├── FD3D12Queue [Compute]  ← ID3D12CommandQueue (Compute)
                          ├── FD3D12Queue [Copy]     ← ID3D12CommandQueue (Copy)
                          ├── FD3D12DescriptorManager← ディスクリプタヒープ管理
                          └── FD3D12BindlessDescriptors ← バインドレスディスクリプタ
```

---

## FD3D12DynamicRHI

```cpp
class FD3D12DynamicRHI : public FDynamicRHI
{
public:
    // モジュール初期化
    virtual void Init() override;
    virtual void Shutdown() override;

    // アダプタ列挙・選択
    void FindAdapter();
    FD3D12Adapter* GetAdapter(uint32 Index = 0);

    // リソース生成（FDynamicRHI の実装）
    virtual FTextureRHIRef RHICreateTexture(
        const FRHITextureCreateDesc& CreateDesc) override;
    virtual FBufferRHIRef  RHICreateBuffer(
        FRHICommandListBase& RHICmdList,
        const FRHIBufferCreateDesc& CreateDesc,
        ERHIAccess InResourceState) override;

    // フレーム制御
    virtual void RHIBeginFrame() override;
    virtual void RHIEndFrame(const FRHIEndFrameArgs& Args) override;

    // PresEnt
    virtual void RHIEndDrawingViewport(
        FRHIViewport* Viewport, bool bPresent, bool bLockToVsync) override;
};
```

---

## FD3D12Adapter

`IDXGIAdapter` に対応する論理ユニット。GPU ごとに存在する。

```cpp
class FD3D12Adapter
{
public:
    // 保有デバイスへのアクセス
    FD3D12Device* GetDevice(uint32 GPUIndex = 0);

    // DXGI オブジェクトへのアクセス
    IDXGIFactory4* GetDXGIFactory4() const;
    IDXGIAdapter*  GetAdapter()      const;

    // D3D12 機能レベル確認
    D3D_FEATURE_LEVEL GetFeatureLevel() const;

    // ルートシグネチャキャッシュ（アダプタ共有）
    FD3D12RootSignatureManager RootSignatureManager;

    // PSO キャッシュ（ディスクキャッシュ対応）
    FD3D12PipelineStateCache   PipelineStateCache;
};
```

---

## FD3D12Device

`ID3D12Device` に対応する核心クラス。GPU リソース・コマンド・ディスクリプタを管理する。

```cpp
class FD3D12Device
{
public:
    // D3D12 ネイティブオブジェクト
    ID3D12Device*  GetDevice()  const;
    ID3D12Device5* GetDevice5() const;  // DXR 対応 (GetDevice5 はレイトレーシング用)

    // コマンドキュー取得
    FD3D12Queue& GetQueue(ED3D12QueueType Type);
    // Type: Graphics / AsyncCompute / Copy

    // ディスクリプタヒープ管理
    FD3D12DescriptorManager& GetDescriptorManager();

    // GPU マスク（マルチ GPU）
    FRHIGPUMask GetGPUMask() const;

    // デバイス遅延削除キュー
    void AddToDeferredDeleteQueue(FD3D12DeferredDeleteObject&& Object);
    void ProcessDeferredDeletionQueue(uint64 CompletedFenceValue);

    // タイミング・プロファイリング
    FD3D12Timing  Timing;

    // ディスクリプタヒープ（タイプ別）
    FD3D12DescriptorHeap*  GetSamplerDescriptorHeap();
    FD3D12DescriptorHeap*  GetGlobalViewDescriptorHeap();  // バインドレス用
};
```

---

## FD3D12Queue

```cpp
class FD3D12Queue
{
public:
    // D3D12 ネイティブキュー
    ID3D12CommandQueue* GetCommandQueue() const;

    // キュータイプ
    ED3D12QueueType GetQueueType() const;
    // ED3D12QueueType::Direct / AsyncCompute / Copy

    // フェンス操作
    uint64 Signal();                    // 現在のフェンス値でシグナル
    void   WaitForFence(uint64 Value);  // CPU を指定フェンスまでブロック
    uint64 GetLastCompletedFence() const;

    // コマンドリストの実行投入
    void ExecuteCommandList(FD3D12CommandList* CommandList);

    // 診断バッファ（DRED 対応）
    FD3D12DiagnosticBuffer DiagnosticBuffer;
};

enum class ED3D12QueueType : uint8
{
    Direct       = 0,  // Graphics + Compute + Copy
    AsyncCompute = 1,  // Compute + Copy (非同期)
    Copy         = 2,  // Copy のみ（テクスチャアップロード等）
};
```

---

## ディスクリプタヒープ管理

D3D12 はシェーダーリソースのバインドに **Descriptor** を使う。  
UE5 の D3D12RHI は2方式を持つ。

### 従来方式（動的ディスクリプタ）

```
FD3D12DescriptorManager
  ├── CBV/SRV/UAV ヒープ（シェーダー可視）
  │     コマンドリスト記録時に必要なディスクリプタをコピーして使用
  └── Sampler ヒープ（シェーダー可視）
```

### バインドレス方式（UE5.3+）

```cpp
class FD3D12BindlessDescriptors
{
    // 全リソースを1つの大きなヒープに登録しておき
    // シェーダーはインデックスで参照
    FD3D12DescriptorHeap GlobalResourceHeap;    // CBV/SRV/UAV 64K エントリ
    FD3D12DescriptorHeap GlobalSamplerHeap;     // Sampler 2K エントリ
};
```

---

## 初期化フロー

```
[起動時]
FD3D12DynamicRHI::Init()
  │
  ├─ CreateDXGIFactory()
  ├─ FD3D12Adapter::EnumerateAdapters()    ← GPU を列挙
  ├─ FD3D12Adapter::InitializeDevices()
  │     └─ FD3D12Device::Initialize()
  │           ├─ ID3D12Device::CreateCommandQueue() × 3   ← Graphics / Compute / Copy
  │           ├─ FD3D12DescriptorManager::Init()
  │           └─ FD3D12BindlessDescriptors::Init()        ← バインドレス対応時
  └─ RootSignatureManager.Init()
```

---

## 関連リファレンス

- [[ref_rhi_d3d12_device]] … FD3D12DynamicRHI / FD3D12Adapter / FD3D12Device / FD3D12Queue の詳細
