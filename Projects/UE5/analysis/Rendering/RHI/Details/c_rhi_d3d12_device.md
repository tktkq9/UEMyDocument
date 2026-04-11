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

## コード実行フロー

```
[起動時 — モジュールロード → デバイス初期化]
FD3D12DynamicRHI::FD3D12DynamicRHI()      D3D12RHI.cpp:60
  └─ FD3D12DynamicRHI::PostInit()          D3D12RHI.cpp:257
       └─ 各アダプタ・デバイスを初期化
            ├─ IDXGIFactory::EnumAdapters1()
            ├─ D3D12CreateDevice()
            ├─ FD3D12Queue::Initialize()   (3種 × GPU 数)
            │    └─ ID3D12Device::CreateCommandQueue()
            └─ FD3D12DescriptorManager::Init()

[フレーム終了 — GPU 投入]
FD3D12DynamicRHI::RHIEndFrame()            D3D12RHI.cpp:572
  └─ FD3D12ContextCommon::CloseCommandList()  D3D12CommandContext.cpp:378
       ├─ FlushResourceBarriers()           D3D12CommandList.cpp:70
       └─ ID3D12GraphicsCommandList::Close()

FD3D12Queue::ExecuteCommandList()
  └─ FD3D12SubmissionThread::ProcessPayload()  D3D12Submission.cpp:553
       └─ ID3D12CommandQueue::ExecuteCommandLists()  D3D12Submission.cpp:827
            └─ ID3D12CommandQueue::Signal(Fence, ++FenceValue)

[フェンス完了後 — リソース解放]
FD3D12Device::ProcessDeferredDeletionQueue(CompletedFenceValue)
  └─ GetLastCompletedFence() >= ObjectFenceValue の全エントリを破棄
       ├─ ID3D12Object::Release()
       └─ FD3D12CommandAllocator::Reset()  ← 再利用プールへ返却

[シャットダウン]
FD3D12DynamicRHI::Shutdown()              D3D12RHI.cpp:298
  └─ 全フェンス完了を待機 → 全リソース解放
```

### 関与クラス・関数一覧

| クラス / 関数 | ファイル:行 | 役割 |
|-------------|-----------|------|
| `FD3D12DynamicRHI::FD3D12DynamicRHI()` | `D3D12RHI.cpp:60` | アダプタ配列を受け取るコンストラクタ |
| `FD3D12DynamicRHI::PostInit()` | `D3D12RHI.cpp:257` | 遅延初期化（RHI スレッド開始後） |
| `FD3D12DynamicRHI::Shutdown()` | `D3D12RHI.cpp:298` | 全リソース解放・デバイス破棄 |
| `FD3D12DynamicRHI::RHIEndFrame()` | `D3D12RHI.cpp:572` | フレーム終了処理・投入トリガー |
| `FD3D12ContextCommon::CloseCommandList()` | `D3D12CommandContext.cpp:378` | コマンドリストをクローズして投入準備 |
| `FD3D12Queue::ExecuteCommandList()` | `D3D12Queue.h` | Submission Thread への Enqueue |
| `ID3D12CommandQueue::ExecuteCommandLists()` | `D3D12Submission.cpp:827` | 実際の GPU 投入 |
| `FD3D12Device::ProcessDeferredDeletionQueue()` | `D3D12Device.cpp` | フェンス完了後の遅延破棄処理 |

---

## 関連リファレンス

- [[ref_rhi_d3d12_device]] … FD3D12DynamicRHI / FD3D12Adapter / FD3D12Device / FD3D12Queue の詳細
