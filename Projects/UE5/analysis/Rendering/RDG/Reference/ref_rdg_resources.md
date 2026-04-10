# REF: RDG リソース型

- 対象ファイル: `RenderGraphResources.h`, `RenderGraphResources.inl`, `RenderGraphDefinitions.h`
- 関連Details: [[b_rdg_resources]]

---

## 型エイリアス一覧

```cpp
// リソースポインタ（= FRDGXxx*）
using FRDGTextureRef        = FRDGTexture*;
using FRDGBufferRef         = FRDGBuffer*;
using FRDGTextureSRVRef     = FRDGShaderResourceView*;  // テクスチャ SRV
using FRDGBufferSRVRef      = FRDGShaderResourceView*;  // バッファ SRV
using FRDGTextureUAVRef     = FRDGUnorderedAccessView*;
using FRDGBufferUAVRef      = FRDGUnorderedAccessView*;

// ハンドル（グラフ内インデックス）
using FRDGTextureHandle        = TRDGHandle<FRDGTexture, uint32>;
using FRDGBufferHandle         = TRDGHandle<FRDGBuffer, uint32>;
using FRDGViewHandle           = TRDGHandle<FRDGView, uint32>;
using FRDGUniformBufferHandle  = TRDGHandle<FRDGUniformBuffer, uint32>;
using FRDGPassHandle           = TRDGHandle<FRDGPass, uint32>;

// 型付き Uniform Buffer
template<typename T>
using TRDGUniformBufferRef = FRDGUniformBuffer*;
```

---

## FRDGResource（全リソースの基底）

```cpp
class FRDGResource
{
public:
    const TCHAR* GetName() const;
    virtual void MarkResourceAsUsed();   // 外部リソースを "使用済み" にマーク
    FRHIResource* GetRHI() const;        // RHI リソース（パス内でのみ有効）
};
```

---

## FRDGViewableResource（テクスチャ / バッファの基底）

```cpp
class FRDGViewableResource : public FRDGResource
{
public:
    const ERDGViewableResourceType Type;  // Texture / Buffer

    bool IsExternal()      const;  // RegisterExternal で登録されたか
    bool IsExtracted()     const;  // QueueExtraction が呼ばれたか
    bool HasBeenProduced() const;  // 書き込みパスが存在するか
    bool IsCulled()        const;  // 参照なしでカリングされたか
    bool IsAccessAllowed(ERHIAccess Access) const;

    void SetOwnerName(const FName& InOwnerName);
};
```

---

## FRDGTexture

```cpp
class FRDGTexture final : public FRDGViewableResource
{
public:
    static const ERDGViewableResourceType StaticType = ERDGViewableResourceType::Texture;

    const FRDGTextureDesc Desc;
    const ERDGTextureFlags Flags;

    FRHITexture*      GetRHI() const;
    FRDGTextureHandle GetHandle() const;
    FRDGTextureSubresourceLayout GetSubresourceLayout() const;
    FRDGTextureSubresourceRange  GetSubresourceRange() const;
    uint32 GetSubresourceCount() const;
};

// FRDGTextureDesc 生成ファクトリ
FRDGTextureDesc FRDGTextureDesc::Create2D(
    FIntPoint Size, EPixelFormat Format, FClearValueBinding ClearValue,
    ETextureCreateFlags Flags, uint8 NumMips = 1, uint8 NumSamples = 1);

FRDGTextureDesc FRDGTextureDesc::Create2DArray(
    FIntPoint Size, EPixelFormat Format, FClearValueBinding ClearValue,
    ETextureCreateFlags Flags, uint16 ArraySize, uint8 NumMips = 1);

FRDGTextureDesc FRDGTextureDesc::Create3D(
    FIntVector Size, EPixelFormat Format, FClearValueBinding ClearValue,
    ETextureCreateFlags Flags, uint8 NumMips = 1);

FRDGTextureDesc FRDGTextureDesc::CreateCube(
    uint32 Size, EPixelFormat Format, FClearValueBinding ClearValue,
    ETextureCreateFlags Flags, uint8 NumMips = 1);

// ERDGTextureFlags
enum class ERDGTextureFlags : uint8 {
    None                       = 0,
    MultiFrame                 = 1 << 0,
    SkipTracking               = 1 << 1,
    ForceImmediateFirstBarrier = 1 << 2,
    MaintainCompression        = 1 << 3,
};
```

---

## FRDGTextureSRVDesc

```cpp
struct FRDGTextureSRVDesc
{
    FRDGTextureRef Texture = nullptr;
    uint8 MipLevel = 0;
    uint8 NumMipLevels = 1;
    EPixelFormat Format = PF_Unknown;
    uint16 FirstArraySlice = 0;
    uint16 NumArraySlices = 0;
    ERDGTextureSRVOverrideFormat FormatOverride = ERDGTextureSRVOverrideFormat::None;

    // ファクトリ
    static FRDGTextureSRVDesc Create(FRDGTextureRef Texture);
    static FRDGTextureSRVDesc CreateForMipLevel(FRDGTextureRef Texture, int32 MipLevel);
    static FRDGTextureSRVDesc CreateWithPixelFormat(FRDGTextureRef Texture, EPixelFormat PixelFormat);
    static FRDGTextureSRVDesc CreateForSlice(FRDGTextureRef Texture, uint32 ArraySlice);
    static FRDGTextureSRVDesc CreateForMetaData(FRDGTextureRef Texture, ERDGTextureMetaDataAccess MetaData);
};
```

---

## FRDGTextureUAVDesc

```cpp
struct FRDGTextureUAVDesc
{
    FRDGTextureRef Texture = nullptr;
    uint8 MipLevel = 0;
    EPixelFormat Format = PF_Unknown;
    uint16 FirstArraySlice = 0;
    uint16 NumArraySlices = 0;

    FRDGTextureUAVDesc(FRDGTextureRef InTexture, uint8 InMipLevel = 0,
        EPixelFormat InFormat = PF_Unknown);
};
```

---

## FRDGBuffer

```cpp
class FRDGBuffer final : public FRDGViewableResource
{
public:
    static const ERDGViewableResourceType StaticType = ERDGViewableResourceType::Buffer;

    const FRDGBufferDesc Desc;
    const ERDGBufferFlags Flags;

    FRHIBuffer*      GetRHI() const;
    FRDGBufferHandle GetHandle() const;
    uint32 GetSize() const;
};

// FRDGBufferDesc 生成ファクトリ
FRDGBufferDesc FRDGBufferDesc::CreateStructuredDesc(uint32 BytesPerElement, uint32 NumElements);
FRDGBufferDesc FRDGBufferDesc::CreateByteAddressDesc(uint32 NumBytes);
FRDGBufferDesc FRDGBufferDesc::CreateBufferDesc(uint32 BytesPerElement, uint32 NumElements);
FRDGBufferDesc FRDGBufferDesc::CreateUploadDesc(uint32 BytesPerElement, uint32 NumElements);
template<typename TCmd>
FRDGBufferDesc FRDGBufferDesc::CreateIndirectDesc(uint32 NumCommands = 1);

// ERDGBufferFlags
enum class ERDGBufferFlags : uint8 {
    None                       = 0,
    MultiFrame                 = 1 << 0,
    SkipTracking               = 1 << 1,
    ForceImmediateFirstBarrier = 1 << 2,
};
```

---

## FRDGBufferSRVDesc

```cpp
struct FRDGBufferSRVDesc
{
    FRDGBufferRef Buffer = nullptr;
    EPixelFormat Format = PF_Unknown;
    uint32 BytesPerElement = 0;

    FRDGBufferSRVDesc(FRDGBufferRef InBuffer);
    FRDGBufferSRVDesc(FRDGBufferRef InBuffer, EPixelFormat InFormat);
};
```

---

## FRDGBufferUAVDesc

```cpp
struct FRDGBufferUAVDesc
{
    FRDGBufferRef Buffer = nullptr;
    EPixelFormat Format = PF_Unknown;
    bool bSupportsAtomicCounter = false;
    bool bSupportsByteBuffer = false;

    FRDGBufferUAVDesc(FRDGBufferRef InBuffer);
    FRDGBufferUAVDesc(FRDGBufferRef InBuffer, EPixelFormat InFormat);
};
```

---

## FRDGView（SRV / UAV の基底）

```cpp
class FRDGView : public FRDGResource
{
public:
    const ERDGViewType Type;  // TextureUAV / TextureSRV / BufferUAV / BufferSRV

    virtual FRDGViewableResource* GetParent() const = 0;
    FRDGViewHandle GetHandle() const;
};

enum class ERDGViewType : uint8 {
    TextureUAV, TextureSRV, BufferUAV, BufferSRV, MAX
};
```

---

## FRDGUniformBuffer（型付き Uniform Buffer）

```cpp
class FRDGUniformBuffer : public FRDGResource
{
public:
    const FRDGParameterStruct ParameterStruct;

    FRHIUniformBuffer* GetRHI() const;
    FRDGUniformBufferHandle GetHandle() const;
    bool IsGlobal() const;  // グローバル Uniform Buffer か
};
```
