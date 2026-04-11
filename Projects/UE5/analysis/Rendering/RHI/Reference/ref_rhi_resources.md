# REF: FRHIResource 系

- 対象ファイル: `Runtime/RHI/Public/RHIResources.h`, `RHIBufferInitializer.h`, `RHITextureInitializer.h`
- 関連Details: [[b_rhi_resources]]

---

## FRHIResource（基底クラス）

```cpp
class FRHIResource
{
public:
    uint32 AddRef()  const;   // 参照カウントをインクリメント
    uint32 Release() const;   // デクリメント→0 で削除キューへ
    uint32 GetRefCount() const;
    bool   IsValid() const;
    void   DisableLifetimeExtension();  // 延長生存を禁止

    ERHIResourceType GetType() const;
    FName            GetOwnerName() const;  // デバッグ用（RHI_ENABLE_RESOURCE_INFO 時のみ）
};

// スマートポインタ型エイリアス
using TRHIResourceRef<T> = TRefCountPtr<T>;
using FTextureRHIRef     = TRefCountPtr<FRHITexture>;
using FBufferRHIRef      = TRefCountPtr<FRHIBuffer>;
```

---

## FRHITexture

```cpp
class FRHITexture : public FRHIResource
{
public:
    // テクスチャ情報
    EPixelFormat      GetFormat()   const;
    uint32            GetSizeX()    const;
    uint32            GetSizeY()    const;
    uint32            GetSizeZ()    const;
    uint32            GetNumMips()  const;
    uint32            GetNumSamples() const;
    ETextureCreateFlags GetFlags()  const;
    FClearValueBinding GetClearBinding() const;

    // ビュー（SRV/RTV/DSV 等）の遅延生成
    FRHIShaderResourceView*   GetOrCreateSRV(const FRHITextureSRVCreateInfo& Info);
};
```

### FRHITextureCreateDesc

```cpp
struct FRHITextureCreateDesc
{
    ETextureDimension Dimension;    // Texture2D / Texture2DArray / Texture3D / TextureCube
    EPixelFormat      Format;
    uint32 SizeX, SizeY, Depth;
    uint32 ArraySize;               // Texture2DArray の枚数
    uint8  NumMips;
    uint8  NumSamples;              // MSAA（1=通常）
    ETextureCreateFlags Flags;
    FClearValueBinding ClearValue;
    const TCHAR* DebugName;

    // ファクトリ
    static FRHITextureCreateDesc Create2D(...);
    static FRHITextureCreateDesc Create2DArray(...);
    static FRHITextureCreateDesc Create3D(...);
    static FRHITextureCreateDesc CreateCube(...);
};
```

---

## FRHIBuffer

UE5.3 以降は `FRHIVertexBuffer` / `FRHIIndexBuffer` を廃止して `FRHIBuffer` に統合。

```cpp
class FRHIBuffer : public FRHIResource
{
public:
    uint32           GetSize()   const;
    uint32           GetStride() const;
    EBufferUsageFlags GetUsage() const;

    // バッファ内容の更新（CPU→GPU コピー）
    void* Lock(FRHICommandListBase& RHICmdList,
               EResourceLockMode LockMode, uint32 Size, bool bDiscard);
    void  Unlock(FRHICommandListBase& RHICmdList);
};
```

### FRHIBufferCreateDesc

```cpp
struct FRHIBufferCreateDesc
{
    const TCHAR*      DebugName;
    uint32            Size;     // バイトサイズ
    uint32            Stride;   // 要素サイズ（Structured Buffer 用）
    EBufferUsageFlags Usage;
    ERHIAccess        InitialState;
    FResourceArrayInterface* ResourceArray;  // 初期データ（nullptr=未初期化）
};
```

### EBufferUsageFlags（主要値）

```cpp
enum class EBufferUsageFlags : uint32
{
    None                 = 0,
    VertexBuffer         = 1 << 0,
    IndexBuffer          = 1 << 1,
    StructuredBuffer     = 1 << 2,
    UniformBuffer        = 1 << 3,  // Constant Buffer
    ByteAddressBuffer    = 1 << 4,  // Raw Buffer（バイトアドレス指定）
    DrawIndirect         = 1 << 5,  // Indirect Draw 引数
    ShaderResource       = 1 << 6,  // SRV としてバインド可能
    UnorderedAccess      = 1 << 7,  // UAV としてバインド可能
    StreamOutput         = 1 << 8,  // Stream Output (SO)
    KeepCPUAccessible    = 1 << 9,  // CPU アクセス保持（ステージングバッファ）
    Shared               = 1 << 10, // クロスデバイス共有
    AccelerationStructure= 1 << 11, // DXR BLAS / TLAS
    Dynamic              = 1 << 12, // CPU から頻繁に更新（Ring Buffer 最適化）
};
```

---

## FRHIUniformBuffer

```cpp
class FRHIUniformBuffer : public FRHIResource
{
public:
    uint32                     GetSize() const;
    const FRHIUniformBufferLayout* GetLayout() const;
};

// 生成
FUniformBufferRHIRef UB = RHICreateUniformBuffer(
    Data,         // CPU データポインタ
    Layout,       // バインディングレイアウト
    UniformBuffer_SingleFrame);  // 寿命（SingleFrame / MultiFrame / SingleDraw）
```

---

## ビュー（SRV / UAV）

```cpp
// SRV 生成記述子
struct FRHIViewDesc
{
    struct FTextureSRV
    {
        EPixelFormat Format;
        uint8 MipLevel;       // 参照開始 MIP
        uint8 NumMipLevels;
        uint16 ArraySlice;
        uint16 NumArraySlices;
        // …
    };
    struct FBufferSRV
    {
        EPixelFormat Format;
        uint32 StartOffsetBytes;
        uint32 NumElements;
        // …
    };
    struct FTextureUAV { /* テクスチャ UAV */ };
    struct FBufferUAV  { /* バッファ UAV  */ };
};

// SRV 生成
FShaderResourceViewRHIRef SRV = RHICreateShaderResourceView(
    Texture, FRHIViewDesc::FTextureSRV{...});
FShaderResourceViewRHIRef BufSRV = RHICreateShaderResourceView(
    Buffer, FRHIViewDesc::FBufferSRV{...});

// UAV 生成
FUnorderedAccessViewRHIRef UAV = RHICreateUnorderedAccessView(
    Texture, 0 /*MipLevel*/);
FUnorderedAccessViewRHIRef BufUAV = RHICreateUnorderedAccessView(
    Buffer, false /*bUseUAVCounter*/, false /*bAppendBuffer*/);
```

---

## PSO 関連リソース

```cpp
// ブレンドステート
FBlendStateRHIRef BlendState = RHICreateBlendState(
    FBlendStateInitializerRHI(
        FBlendStateInitializerRHI::FRenderTarget(
            true, // bEnableBlend
            BF_One, BF_InverseSourceAlpha,    // Color
            BO_Add,
            BF_One, BF_InverseSourceAlpha,    // Alpha
            BO_Add)));

// ラスタライザステート
FRasterizerStateRHIRef RastState = RHICreateRasterizerState(
    FRasterizerStateInitializerRHI(FM_Solid, CM_CW, 0, 0, false));

// 深度ステンシルステート
FDepthStencilStateRHIRef DSState = RHICreateDepthStencilState(
    FDepthStencilStateInitializerRHI(true, true, CF_GreaterEqual));

// サンプラー
FSamplerStateRHIRef Sampler = RHICreateSamplerState(
    FSamplerStateInitializerRHI(SF_Bilinear, AM_Clamp, AM_Clamp, AM_Clamp));
```
