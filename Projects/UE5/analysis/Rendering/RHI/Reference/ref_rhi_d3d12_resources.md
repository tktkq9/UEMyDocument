# REF: D3D12 リソース系クラス

- 対象ファイル: `D3D12RHI/Private/D3D12Resources.h/.cpp`, `D3D12Buffer.cpp`, `D3D12Texture.h/.cpp`, `D3D12Descriptors.h/.cpp`, `D3D12View.h`
- 関連Details: [[b_rhi_resources]] | [[c_rhi_d3d12_device]]

---

## FD3D12Resource（D3D12 リソース基底）

```cpp
class FD3D12Resource
{
public:
    // D3D12 ネイティブ
    ID3D12Resource* GetResource() const;

    // GPU 仮想アドレス（バッファ / AccelerationStructure 用）
    D3D12_GPU_VIRTUAL_ADDRESS GetGPUVirtualAddress() const;

    // リソース記述子
    const D3D12_RESOURCE_DESC& GetDesc() const;

    // 現在のリソース状態
    D3D12_RESOURCE_STATES GetDefaultResourceState() const;

    // ヒープ情報
    FD3D12Heap*      GetHeap()       const;
    D3D12_HEAP_TYPE  GetHeapType()   const;
    // D3D12_HEAP_TYPE_DEFAULT  … VRAM 専用
    // D3D12_HEAP_TYPE_UPLOAD   … CPU→GPU アップロード（Write Combine）
    // D3D12_HEAP_TYPE_READBACK … GPU→CPU リードバック

    // デバッグ名
    void SetName(const TCHAR* Name);
};
```

---

## FD3D12Buffer

`FRHIBuffer` の D3D12 実装。`FD3D12Resource` を内包する。

```cpp
class FD3D12Buffer : public FRHIBuffer
{
public:
    // ネイティブリソースへのアクセス
    FD3D12Resource* GetResource() const;

    // GPU 仮想アドレス（IndirectDraw / ConstantBuffer 用）
    D3D12_GPU_VIRTUAL_ADDRESS GetGPUVirtualAddress() const;

    // ロック（CPU アクセス）
    void* Lock(uint32 Offset, uint32 Size, EResourceLockMode LockMode);
    void  Unlock();

    // SRV / UAV の遅延生成
    FD3D12ShaderResourceView*  GetOrCreateSRV(const FRHIViewDesc& Desc);
    FD3D12UnorderedAccessView* GetOrCreateUAV(const FRHIViewDesc& Desc);
};
```

### バッファアロケーション戦略

```
BUF_Dynamic (CPU 頻繁更新):
    → FD3D12UploadHeapAllocator（リングバッファ、UPLOAD ヒープ）
    → Map/Unmap ではなく毎フレーム新しい領域を使用

BUF_Static (静的データ):
    → FD3D12DefaultBufferAllocator（サブアロケーション、DEFAULT ヒープ）
    → アップロード済み後はステージングバッファを破棄

BUF_Structured / UAV:
    → FD3D12DefaultBufferAllocator（個別リソース、DEFAULT ヒープ）
```

---

## FD3D12Texture

`FRHITexture` の D3D12 実装。

```cpp
class FD3D12Texture : public FRHITexture
{
public:
    // ネイティブリソース
    FD3D12Resource* GetResource() const;

    // RTV / DSV / SRV へのアクセス
    FD3D12RenderTargetView*        GetRenderTargetView(int32 MipIndex, int32 ArraySliceIndex) const;
    FD3D12DepthStencilView*        GetDepthStencilView(FExclusiveDepthStencil AccessType) const;
    FD3D12ShaderResourceView*      GetShaderResourceView() const;

    // MSAA テクスチャのリゾルブ
    FD3D12Resource* GetResolveTarget() const;
};
```

### テクスチャメモリレイアウト

```
テクスチャ作成フラグ → D3D12_HEAP_TYPE の選択

TexCreate_RenderTargetable | TexCreate_ShaderResource:
    → D3D12_HEAP_TYPE_DEFAULT
    → D3D12_RESOURCE_STATE_RENDER_TARGET （初期状態）

TexCreate_CPUReadback:
    → D3D12_HEAP_TYPE_READBACK
    → D3D12_RESOURCE_STATE_COPY_DEST

TexCreate_Dynamic（CPU 更新頻繁）:
    → D3D12_HEAP_TYPE_UPLOAD
```

---

## FD3D12Heap

```cpp
class FD3D12Heap
{
public:
    ID3D12Heap* GetHeap() const;

    uint64 GetSize()        const;
    D3D12_HEAP_TYPE GetType() const;

    // ヒープ内にリソースを配置（Place）
    FD3D12Resource* CreatePlacedResource(
        const D3D12_RESOURCE_DESC& ResourceDesc,
        D3D12_RESOURCE_STATES InitialState,
        uint64 HeapOffset);
};
```

---

## Descriptor 系クラス

### FD3D12ShaderResourceView

```cpp
class FD3D12ShaderResourceView : public FRHIShaderResourceView
{
public:
    // CPU / GPU ディスクリプタハンドル
    D3D12_CPU_DESCRIPTOR_HANDLE GetCPUHandle() const;
    D3D12_GPU_DESCRIPTOR_HANDLE GetGPUHandle() const;  // シェーダー可視ヒープ上

    // バインドレス用インデックス（PLATFORM_SUPPORTS_BINDLESS_RENDERING 時のみ）
    uint32 GetBindlessIndex() const;
};
```

### FD3D12UnorderedAccessView

```cpp
class FD3D12UnorderedAccessView : public FRHIUnorderedAccessView
{
public:
    D3D12_CPU_DESCRIPTOR_HANDLE GetCPUHandle()    const;
    D3D12_CPU_DESCRIPTOR_HANDLE GetCounterCPUHandle() const;  // UAV Counter 付きバッファ用
    uint32 GetBindlessIndex() const;
};
```

### FD3D12RenderTargetView

```cpp
class FD3D12RenderTargetView
{
public:
    D3D12_CPU_DESCRIPTOR_HANDLE GetCPUHandle() const;
    FD3D12Texture* GetResource() const;
    D3D12_RENDER_TARGET_VIEW_DESC GetDesc() const;
};
```

### FD3D12DepthStencilView

```cpp
class FD3D12DepthStencilView
{
public:
    D3D12_CPU_DESCRIPTOR_HANDLE GetCPUHandle() const;
    // アクセスタイプ（Depth書き込み可 / Stencil書き込み可 の組み合わせ）
    FExclusiveDepthStencil GetDepthStencilAccess() const;
};
```

---

## バインドレスディスクリプタ（UE5.3+）

```cpp
class FD3D12BindlessDescriptors
{
public:
    // グローバルヒープ（全 SRV / UAV / CBV を登録）
    FD3D12DescriptorHeap GlobalResourceHeap;  // 最大 D3D12_MAX_SHADER_VISIBLE_DESCRIPTOR_HEAP_SIZE_TIER_2 (1M) エントリ
    FD3D12DescriptorHeap GlobalSamplerHeap;   // 最大 D3D12_MAX_SHADER_VISIBLE_SAMPLER_HEAP_SIZE (2048) エントリ

    // リソース登録 → インデックス取得
    FRHIDescriptorHandle AllocateDescriptor(
        D3D12_CPU_DESCRIPTOR_HANDLE SourceCPUHandle,
        D3D12_DESCRIPTOR_HEAP_TYPE HeapType);

    void FreeDescriptor(FRHIDescriptorHandle Handle);
};
```

---

## FD3D12PipelineState（PSO）

```cpp
class FD3D12PipelineState
{
public:
    // D3D12 ネイティブ PSO
    ID3D12PipelineState* GetPipelineState() const;

    // 対応ルートシグネチャ
    FD3D12RootSignature* GetRootSignature() const;

    // タイプ判定
    bool IsGraphics() const;
    bool IsCompute()  const;
    bool IsRayTracing() const;
};
```

### PSO キャッシュ

```cpp
class FD3D12PipelineStateCache
{
    // メモリキャッシュ（LRU）
    TMap<FD3D12HighLevelGraphicsPipelineStateDesc, FD3D12PipelineState*> GraphicsMap;
    TMap<FD3D12ComputePipelineStateDesc,           FD3D12PipelineState*> ComputeMap;

    // ディスクキャッシュ（D3D12 PSO library）
    ID3D12PipelineLibrary* PSOLibrary;   // シェーダーバイナリキャッシュ

    // 取得または生成
    FD3D12PipelineState* FindOrAdd(
        const FGraphicsPipelineStateInitializer& Initializer);
    FD3D12PipelineState* FindOrAdd(
        const FRHIComputeShader* ComputeShader);
};
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.D3D12.PSOFileCacheDirectory` | "" | PSO ディスクキャッシュ保存先 |
| `r.D3D12.EnablePSOCache` | 1 | PSO キャッシュ有効 |
| `r.D3D12.SubAllocatorBufferPoolSize` | 8MB | バッファサブアロケータプールサイズ |
| `r.D3D12.TransientResourceAllocator` | 1 | Transient リソースアロケータ有効 |

---

> [!note]- FD3D12Resource の Placed / Committed / Reserved の使い分け
> D3D12 では3種のリソース配置方式がある。**Committed** (`CreateCommittedResource`) はヒープとリソースを同時作成する最もシンプルな方式。  
> **Placed** (`CreatePlacedResource`) は事前確保した `FD3D12Heap` 上に配置し、同じヒープで複数リソースをエイリアシングできる（Transient リソースはこれを使う）。  
> UE5 の Transient Resource Allocator（`r.D3D12.TransientResourceAllocator=1`）は Placed を使い、生存期間が重ならないリソース間でメモリを再利用することで VRAM 消費を削減する。

> [!note]- FD3D12PipelineStateCache のディスクキャッシュと初回ロード
> `FD3D12PipelineStateCache` は `ID3D12PipelineLibrary`（D3D12 PSO Library）を使い、シェーダーバイナリとパイプライン設定をディスクに保存する。  
> `r.D3D12.EnablePSOCache=1` かつ `r.D3D12.PSOFileCacheDirectory` にパスを設定すると、次回起動時にキャッシュから PSO を復元でき、ドライバーによる再コンパイルを回避できる。  
> キャッシュミス時は `ID3D12Device::CreateGraphicsPipelineState()` が非同期で実行され（`r.D3D12.AsyncPSOCreation=1` の場合）、完了まで前の PSO をフォールバックとして使う。

> [!note]- バインドレスディスクリプタと従来のディスクリプタコピー方式の違い
> 従来方式では描画ごとに CPU 側のオフラインヒープからシェーダー可視ヒープへ必要なディスクリプタをコピーする（`FD3D12ExplicitDescriptorCache`）。これはドローコール数に比例してオーバーヘッドが増える。  
> バインドレス方式（`r.D3D12.BindlessResources=1`）では全リソースを `FD3D12BindlessDescriptors::GlobalResourceHeap`（最大 1M エントリ）に登録し、シェーダーはインデックスで参照する。ディスクリプタコピーが不要になりドローコールのCPUコストが大幅に下がる。  
> バインドレスは SM6+ かつ `D3D12_RESOURCE_BINDING_TIER_3` 以上のハードウェアが必要。UE5 では対応 GPU でのみ自動選択される。
