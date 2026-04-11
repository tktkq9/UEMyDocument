# RHI リソース管理

- 対象: `Runtime/RHI/Public/RHIResources.h`, `RHIPipeline.h`, `RHIShaderParameters.h`
- 関連Reference: [[ref_rhi_resources]]

---

## 概要

RHI リソースは `FRHIResource` を基底とし、**参照カウント（TRefCountPtr）** で生存期間を管理する。  
レンダラーが保持するのは `TRefCountPtr<FRHITexture>` 等のスマートポインタであり、  
参照がゼロになると削除キューに入り、GPU フェンス完了後に安全に破棄される。

---

## リソース継承ツリー

```
FRHIResource                          ← 基底（参照カウント・削除キュー管理）
  ├── FRHITexture                     ← 2D / 3D / Cube / Array テクスチャ共通
  │     └── FRHITextureReference      ← 間接参照（実テクスチャを後から差し替え可能）
  ├── FRHIBuffer                      ← VertexBuffer / IndexBuffer / UniformBuffer 等共通
  │     ├── FRHIUniformBuffer         ← Uniform / Constant バッファ
  │     └── (Vertex / Index は FRHIBuffer で統合済み UE5.3 以降)
  ├── FRHIShader 系
  │     ├── FRHIVertexShader
  │     ├── FRHIPixelShader
  │     ├── FRHIComputeShader
  │     └── FRHIMeshShader 等
  ├── FRHIShaderResourceView (SRV)    ← テクスチャ / バッファの読み取りビュー
  ├── FRHIUnorderedAccessView (UAV)   ← テクスチャ / バッファの読み書きビュー
  ├── FRHISamplerState                ← サンプラー設定
  ├── FRHIGraphicsPipelineState       ← Graphics PSO（VS/PS/ブレンド/ラスタ等）
  ├── FRHIComputePipelineState        ← Compute PSO
  └── FRHIRenderQuery                 ← オクルージョン / タイムスタンプクエリ
```

---

## FRHIResource

```cpp
class FRHIResource
{
public:
    // 参照カウント操作
    uint32 AddRef()  const;
    uint32 Release() const;    // 0 になると削除キューへ
    uint32 GetRefCount() const;
    bool   IsValid() const;

    // リソース種別
    ERHIResourceType GetType() const;

    // 所有者名（デバッグ用）
    FName GetOwnerName() const;  // RHI_ENABLE_RESOURCE_INFO が有効時のみ

private:
    // 削除キュー（GPU フェンス完了後に破棄）
    static void DeleteResources(TArray<FRHIResource*> const& Resources);
};
```

---

## FRHITexture

```cpp
// テクスチャ生成記述子
struct FRHITextureCreateDesc
{
    FRHITextureCreateDesc& SetDimension(ETextureDimension InDimension);
    FRHITextureCreateDesc& SetFormat(EPixelFormat InFormat);
    FRHITextureCreateDesc& SetExtent(uint32 InSizeX, uint32 InSizeY);
    FRHITextureCreateDesc& SetDepth(uint32 InDepth);
    FRHITextureCreateDesc& SetNumMips(uint8 InNumMips);
    FRHITextureCreateDesc& SetNumSamples(uint8 InNumSamples);
    FRHITextureCreateDesc& SetFlags(ETextureCreateFlags InFlags);
    FRHITextureCreateDesc& SetClearValue(FClearValueBinding InClearValue);
    FRHITextureCreateDesc& SetDebugName(const TCHAR* InDebugName);

    // 静的ファクトリ
    static FRHITextureCreateDesc Create2D(const TCHAR* Name, uint32 X, uint32 Y, EPixelFormat Format);
    static FRHITextureCreateDesc Create3D(const TCHAR* Name, uint32 X, uint32 Y, uint32 Z, EPixelFormat Format);
    static FRHITextureCreateDesc CreateCube(const TCHAR* Name, uint32 Size, EPixelFormat Format);
};

// 生成
FTextureRHIRef Texture = RHICreateTexture(Desc);
```

### 主要フラグ（ETextureCreateFlags）

| フラグ | 説明 |
|--------|------|
| `TexCreate_RenderTargetable` | レンダーターゲットとして使用 |
| `TexCreate_ShaderResource` | SRV としてシェーダーから参照 |
| `TexCreate_UAV` | UAV として Compute シェーダーから書き込み |
| `TexCreate_DepthStencilTargetable` | 深度/ステンシルバッファとして使用 |
| `TexCreate_ResolveTargetable` | MSAA リゾルブターゲット |
| `TexCreate_Transient` | フレーム内のみ有効（メモリ効率化）|
| `TexCreate_SRGB` | sRGB ガンマ適用 |

---

## FRHIBuffer

UE5.3 以降、`FRHIVertexBuffer` / `FRHIIndexBuffer` / `FRHIStructuredBuffer` は  
すべて `FRHIBuffer` に統合された。`EBufferUsageFlags` で用途を区別する。

```cpp
struct FRHIBufferCreateDesc
{
    const TCHAR* DebugName;
    uint32 Size;                    // バイトサイズ
    uint32 Stride;                  // 要素ストライド（Structured Buffer 用）
    EBufferUsageFlags Usage;        // 用途フラグ
    ERHIAccess InitialState;        // 初期アクセス状態
    FResourceArrayInterface* ResourceArray; // 初期データ
};

// 生成
FBufferRHIRef Buffer = RHICreateBuffer(RHICmdList, Desc, ERHIAccess::VertexOrIndexBuffer);
```

### 主要フラグ（EBufferUsageFlags）

| フラグ | 説明 |
|--------|------|
| `BUF_VertexBuffer` | 頂点バッファ |
| `BUF_IndexBuffer` | インデックスバッファ |
| `BUF_UniformBuffer` | ユニフォームバッファ（CBV）|
| `BUF_StructuredBuffer` | 構造化バッファ（SRV / UAV）|
| `BUF_ByteAddressBuffer` | バイトアドレスバッファ（Raw Buffer）|
| `BUF_DrawIndirect` | IndirectDraw 引数バッファ |
| `BUF_ShaderResource` | SRV としてバインド可能 |
| `BUF_UnorderedAccess` | UAV としてバインド可能 |
| `BUF_Dynamic` | CPU から頻繁に更新（Ring Buffer 最適化）|

---

## PSO（パイプラインステートオブジェクト）

### FGraphicsPipelineStateInitializer

```cpp
struct FGraphicsPipelineStateInitializer
{
    // シェーダーバインド
    FRHIVertexDeclaration*  BoundShaderState.VertexDeclarationRHI;
    FRHIVertexShader*       BoundShaderState.VertexShaderRHI;
    FRHIPixelShader*        BoundShaderState.PixelShaderRHI;
    FRHIMeshShader*         BoundShaderState.MeshShaderRHI;      // Mesh Shader
    FRHIAmplificationShader* BoundShaderState.AmplificationShaderRHI;

    // 固定機能ステート
    FRHIBlendState*         BlendState;
    FRHIRasterizerState*    RasterizerState;
    FRHIDepthStencilState*  DepthStencilState;

    // レンダーターゲット設定
    TStaticArray<EPixelFormat, MaxSimultaneousRenderTargets> RenderTargetFormats;
    EPixelFormat            DepthStencilTargetFormat;
    uint32                  NumSamples;                          // MSAA サンプル数

    // プリミティブ設定
    EPrimitiveType          PrimitiveType;
};

// 取得（PSO キャッシュ経由）
FGraphicsPipelineStateRHIRef PSO = GetAndOrCreateGraphicsPipelineState(
    RHICmdList, Initializer, EApplyRendertargetOption::CheckApply);
```

---

## ERHIAccess（リソースアクセス状態）

```cpp
enum class ERHIAccess
{
    None                  = 0,
    Present               = 1 << 0,   // Present 用
    IndirectArgs          = 1 << 1,   // IndirectDraw 引数
    VertexOrIndexBuffer   = 1 << 2,   // VB / IB
    SRVGraphics           = 1 << 3,   // Graphics シェーダー SRV
    SRVCompute            = 1 << 4,   // Compute シェーダー SRV
    UAVGraphics           = 1 << 5,   // Graphics シェーダー UAV
    UAVCompute            = 1 << 6,   // Compute シェーダー UAV
    RTV                   = 1 << 7,   // レンダーターゲット
    DSVRead               = 1 << 8,   // 深度ステンシル読み取り
    DSVWrite              = 1 << 9,   // 深度ステンシル書き込み
    CopySrc               = 1 << 10,  // コピー元
    CopyDest              = 1 << 11,  // コピー先
    ResolveSrc            = 1 << 12,  // MSAA リゾルブ元
    ResolveDst            = 1 << 13,  // MSAA リゾルブ先
    // …
};
```

RDG はこれを使って自動バリアを生成する（[[e_rhi_rdg_bridge]] 参照）。

---

## 関連リファレンス

- [[ref_rhi_resources]] … FRHIResource / FRHITexture / FRHIBuffer / FRHIUniformBuffer の詳細
