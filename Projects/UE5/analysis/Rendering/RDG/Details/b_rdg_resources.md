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

---

## コード実行フロー

### エントリポイント

```
──── リソース宣言フェーズ（AddPass 前） ────

GraphBuilder.CreateTexture(Desc, Name)
  └─ FRDGTexture を Allocator で確保（宣言のみ・GPU メモリ未確保）
     IF_RDG_ENABLE_TRACE → Trace.AddResource(Texture)
     IF_RDG_ENABLE_DEBUG → UserValidation.ValidateCreateTexture(...)

GraphBuilder.RegisterExternalTexture(PooledRT, Name)
  └─ FRDGTexture を外部テクスチャにバインドして登録
     IPooledRenderTarget::GetRenderTargetItem().TargetableTexture を保持
     MultiFrame フラグなしは Execute() 後に Pool に返却

GraphBuilder.CreateSRV / CreateUAV
  └─ FRDGShaderResourceView / FRDGUnorderedAccessView を宣言

GraphBuilder.QueueTextureExtraction(RDGTexture, &OutPtr)
  └─ EpiloguePass の依存として登録（Execute 後に OutPtr へ書き戻される）

──── Execute() 内リソース確保フェーズ ────

FRDGBuilder::Execute()
  │
  ├─ Compile()
  │   └─ SetupPassDependencies()
  │       └─ 各リソースの FirstPass / LastPass を確定（カリング判定の基礎）
  │
  ├─ AllocatePooledTextures()
  │   └─ CreateTexture() で作成した非 Transient テクスチャを FRenderTargetPool から確保
  │
  ├─ AllocateTransientResources()
  │   └─ Transient テクスチャ／バッファを FRHITransientResourceAllocator でエイリアシング確保
  │      同フレーム内で LastConsumer に達したリソースは即座に解放され再利用される
  │
  └─ ExecutePasses() → ExecutePassEpilogue()
      └─ LastConsumer に達したリソースを Pool に返却
         QueueTextureExtraction の OutPtr に pooled テクスチャを書き戻し
```

### フロー詳細

1. **CreateTexture() — 宣言のみ**
   ```cpp
   FRDGTextureRef Tex = GraphBuilder.CreateTexture(
       FRDGTextureDesc::Create2D(...), TEXT("MyTex"));
   // この時点: FRDGTexture ノードのみ生成。FRHITexture* = nullptr
   // GPU メモリ確保は Execute() 内の AllocatePooledTextures / AllocateTransientResources
   ```
   - `ERDGTextureFlags::MultiFrame` が付いていないテクスチャは Transient 扱いになり得る
   - `r.RDG.TransientResourceAllocator=1` の場合は Transient として確保される

2. **RegisterExternalTexture() — 外部 RHI テクスチャの統合**
   ```cpp
   FRDGTextureRef ExTex = GraphBuilder.RegisterExternalTexture(
       SceneRenderTargets.SceneColor, ERDGTextureFlags::MultiFrame);
   // IPooledRenderTarget が保持する FRHITexture* を RDG に紐付ける
   // FRDGTexture::PooledRenderTarget に IPooledRenderTarget の参照を保持
   ```
   - `MultiFrame` フラグ付きは Execute() 後もアロケータが解放しない
   - `SkipTracking` フラグ付きはバリア計算をスキップ（読み取り専用外部テクスチャに使用）

3. **QueueTextureExtraction() → Execute() 後の書き戻し**
   ```cpp
   TRefCountPtr<IPooledRenderTarget> ExtractedRT;
   GraphBuilder.QueueTextureExtraction(RDGTex, &ExtractedRT);

   GraphBuilder.Execute();
   // ↑ Execute() 完了後、ExtractedRT.IsValid() == true
   // EpiloguePass が LastConsumer となり pooled テクスチャを OutPtr に書き戻す
   ```

4. **Transient リソースのエイリアシング**
   ```
   フレーム内のメモリ使用:
     Pass A → [Tex1 確保] → Pass A 実行 → [Tex1 解放] → [Tex2 が Tex1 のメモリを再利用]
   
   r.RDG.Debug.ExtendResourceLifetimes=1 にするとエイリアシングが無効化され
   全リソースが同時確保される → 最大メモリ使用量の調査に使う
   ```

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|--------|------|
| `FRDGBuilder::CreateTexture()` | `RenderGraphBuilder.h` | テクスチャノードの宣言 |
| `FRDGBuilder::RegisterExternalTexture()` | `RenderGraphBuilder.h` | 外部テクスチャの統合 |
| `FRDGBuilder::QueueTextureExtraction()` | `RenderGraphBuilder.h` | Execute 後の書き戻し登録 |
| `FRDGBuilder::AllocatePooledTextures()` | `RenderGraphBuilder.cpp` | Execute 時のプール確保 |
| `FRDGBuilder::AllocateTransientResources()` | `RenderGraphBuilder.cpp` | Transient エイリアシング確保 |
| `FRDGTexture` | `RenderGraphResources.h` | [[ref_rdg_resources]] テクスチャノード |
| `FRDGBuffer` | `RenderGraphResources.h` | [[ref_rdg_resources]] バッファノード |
| `FRHITransientResourceAllocator` | `RHITransientResourceAllocator.h` | Transient メモリ管理 |
| `FRenderTargetPool` | `RenderTargetPool.h` | [[ref_rdg_allocator]] テクスチャプール |
