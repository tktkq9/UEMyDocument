# B: RDG リソース型

- 対象: `RenderGraphResources.h/.inl`, `RenderGraphDefinitions.h`
- 上位: [[10_rdg_overview]]
- Reference: [[ref_rdg_resources]]

---

## リソース階層

```
FRDGResource
├─ FRDGViewableResource          ← テクスチャ / バッファの基底
│  ├─ FRDGTexture                ← 2D/3D/Cube テクスチャ
│  └─ FRDGBuffer                 ← 構造体 / バイト / 描画引数バッファ
├─ FRDGView                      ← SRV / UAV の基底
│  ├─ FRDGShaderResourceView     ← SRV（Texture / Buffer）
│  └─ FRDGUnorderedAccessView    ← UAV（Texture / Buffer）
└─ FRDGUniformBuffer             ← Uniform Buffer（型付き）
```

---

## FRDGTexture

```cpp
class FRDGTexture final : public FRDGViewableResource
{
public:
    const FRDGTextureDesc Desc;      // 解像度・フォーマット・フラグ
    const ERDGTextureFlags Flags;

    FRHITexture*     GetRHI() const;     // 実際の RHI テクスチャ（パス内でのみ有効）
    FRDGTextureHandle GetHandle() const; // グラフ内ハンドル
};
```

### FRDGTextureDesc — 生成ファクトリ

```cpp
// 2D テクスチャ
FRDGTextureDesc::Create2D(
    FIntPoint Size,
    EPixelFormat Format,
    FClearValueBinding ClearValue,
    ETextureCreateFlags Flags,
    uint8 NumMips = 1,
    uint8 NumSamples = 1);

// 2D テクスチャ配列
FRDGTextureDesc::Create2DArray(FIntPoint Size, ..., uint16 ArraySize);

// 3D テクスチャ
FRDGTextureDesc::Create3D(FIntVector Size, ...);

// Cube マップ
FRDGTextureDesc::CreateCube(uint32 Size, ...);
```

### ERDGTextureFlags

```cpp
enum class ERDGTextureFlags : uint8 {
    None                      = 0,
    MultiFrame                = 1 << 0,  // フレームをまたいで保持（RegisterExternal 併用）
    SkipTracking              = 1 << 1,  // 読み取り専用・バリアトラッキング除外
    ForceImmediateFirstBarrier = 1 << 2, // 最初のバリアを即座に発行
    MaintainCompression       = 1 << 3,  // メタデータ圧縮を維持（帯域節約）
};
```

---

## FRDGBuffer

```cpp
class FRDGBuffer final : public FRDGViewableResource
{
public:
    const FRDGBufferDesc Desc;
    const ERDGBufferFlags Flags;

    FRHIBuffer*     GetRHI() const;
    FRDGBufferHandle GetHandle() const;
};
```

### FRDGBufferDesc — 生成ファクトリ

```cpp
// 構造体バッファ（ShaderResourceView / UAV 使用可）
FRDGBufferDesc::CreateStructuredDesc(uint32 BytesPerElement, uint32 NumElements);

// バイトアドレスバッファ（ByteAddressBuffer / RWByteAddressBuffer）
FRDGBufferDesc::CreateByteAddressDesc(uint32 NumBytes);

// フォーマット付きバッファ（Buffer<float4> 等）
FRDGBufferDesc::CreateBufferDesc(uint32 BytesPerElement, uint32 NumElements);

// 描画間接引数バッファ（DrawIndirect / DispatchIndirect）
FRDGBufferDesc::CreateIndirectDesc<TCmd>(uint32 NumCommands = 1);
// 例: FRDGBufferDesc::CreateIndirectDesc<FRHIDispatchIndirectParameters>(1)

// Upload バッファ（CPU → GPU 転送専用）
FRDGBufferDesc::CreateUploadDesc(uint32 BytesPerElement, uint32 NumElements);
```

### ERDGBufferFlags

```cpp
enum class ERDGBufferFlags : uint8 {
    None                      = 0,
    MultiFrame                = 1 << 0,  // フレームをまたいで保持
    SkipTracking              = 1 << 1,  // 読み取り専用
    ForceImmediateFirstBarrier = 1 << 2,
};
```

---

## View 型（SRV / UAV）

### SRV（Shader Resource View）

```cpp
// テクスチャ SRV
FRDGTextureSRVRef SRV = GraphBuilder.CreateSRV(
    FRDGTextureSRVDesc::Create(Texture));              // 全 Mip
FRDGTextureSRVRef SRV = GraphBuilder.CreateSRV(
    FRDGTextureSRVDesc::CreateForMipLevel(Texture, 2)); // Mip 2 のみ

// バッファ SRV（フォーマット付き）
FRDGBufferSRVRef BufSRV = GraphBuilder.CreateSRV(
    FRDGBufferSRVDesc(Buffer, PF_R32_FLOAT));

// バッファ SRV（構造体）
FRDGBufferSRVRef StructSRV = GraphBuilder.CreateSRV(
    FRDGBufferSRVDesc(Buffer));  // BytesPerElement は Desc から自動
```

### UAV（Unordered Access View）

```cpp
// テクスチャ UAV（Mip 0）
FRDGTextureUAVRef TexUAV = GraphBuilder.CreateUAV(Texture);

// テクスチャ UAV（特定 Mip）
FRDGTextureUAVRef MipUAV = GraphBuilder.CreateUAV(
    FRDGTextureUAVDesc(Texture, /* MipLevel */ 2));

// バッファ UAV（フォーマット付き）
FRDGBufferUAVRef BufUAV = GraphBuilder.CreateUAV(
    FRDGBufferUAVDesc(Buffer, PF_R32_UINT));

// バッファ UAV（構造体）
FRDGBufferUAVRef StructUAV = GraphBuilder.CreateUAV(
    FRDGBufferUAVDesc(Buffer));
```

---

## FRDGViewableResource 共通 API

```cpp
bool IsExternal()      const; // 外部登録リソースか
bool IsExtracted()     const; // QueueExtraction が呼ばれているか
bool HasBeenProduced() const; // 少なくとも 1 つのパスが書き込んだか
bool IsCulled()        const; // 参照パスがなくカリングされたか
```

---

## リソースのライフタイム

```
グラフ内のリソースライフタイム:
  CreateTexture() → [最初に参照するパス] → [最後に参照するパス] → GPU メモリ解放
  
  MultiFrame フラグ付きは Execute() 後も生存（次フレームで RegisterExternal 可能）
  Transient リソースは同フレーム内でエイリアシング（メモリ再利用）される
```

---

## ハンドル型一覧

| 型エイリアス | 指す対象 |
|-------------|---------|
| `FRDGTextureRef` | `FRDGTexture*` |
| `FRDGBufferRef` | `FRDGBuffer*` |
| `FRDGTextureSRVRef` | `FRDGShaderResourceView*`（テクスチャ） |
| `FRDGBufferSRVRef` | `FRDGShaderResourceView*`（バッファ） |
| `FRDGTextureUAVRef` | `FRDGUnorderedAccessView*`（テクスチャ） |
| `FRDGBufferUAVRef` | `FRDGUnorderedAccessView*`（バッファ） |
| `TRDGUniformBufferRef<T>` | `FRDGUniformBuffer*`（型付き） |

---

## 関連リファレンス

| リファレンス | 対象ソース |
|------------|----------|
| [[ref_rdg_resources]] | `RenderGraphResources.h/.inl/.cpp` |
| [[ref_rdg_texture_subresource]] | `RenderGraphTextureSubresource.h` |
| [[ref_rdg_allocator]] | `RenderGraphAllocator.h/.cpp`, `RenderGraphResourcePool.cpp` |
