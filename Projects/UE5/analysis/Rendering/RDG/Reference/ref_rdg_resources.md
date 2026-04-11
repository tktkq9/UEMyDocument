# リファレンス：RenderGraphResources.h / RenderGraphResources.inl / RenderGraphResources.cpp

- グループ: b - Resources
- 上位: [[b_rdg_resources]]
- 関連: [[ref_rdg_texture_subresource]] | [[ref_rdg_allocator]] | [[ref_rdg_builder]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphResources.h`
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphResources.inl`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphResources.cpp`

## 概要

RDG 管理リソース型（テクスチャ・バッファ・View・Uniform Buffer）の全定義。  
型ヒエラルキー:

```
FRDGResource
├─ FRDGViewableResource          ← テクスチャ / バッファの基底
│   ├─ FRDGTexture               ← 2D/3D/Cube テクスチャ
│   └─ FRDGBuffer                ← 構造体 / バイト / DrawIndirect バッファ
├─ FRDGView                      ← SRV / UAV の基底
│   ├─ FRDGShaderResourceView    ← テクスチャ SRV / バッファ SRV
│   └─ FRDGUnorderedAccessView   ← テクスチャ UAV / バッファ UAV
└─ FRDGUniformBuffer             ← 型付き Uniform Buffer（TRDGUniformBuffer<T>）
```

---

## 型エイリアス一覧

```cpp
// リソースポインタ（= FRDGXxx*）
using FRDGTextureRef        = FRDGTexture*;
using FRDGBufferRef         = FRDGBuffer*;
using FRDGTextureSRVRef     = FRDGTextureSRV*;       // テクスチャ SRV（= FRDGShaderResourceView*）
using FRDGBufferSRVRef      = FRDGBufferSRV*;        // バッファ SRV（= FRDGShaderResourceView*）
using FRDGTextureUAVRef     = FRDGTextureUAV*;       // テクスチャ UAV（= FRDGUnorderedAccessView*）
using FRDGBufferUAVRef      = FRDGBufferUAV*;        // バッファ UAV（= FRDGUnorderedAccessView*）
using FRDGUniformBufferRef  = FRDGUniformBuffer*;

// ハンドル（グラフ内インデックス）
using FRDGTextureHandle       = TRDGHandle<FRDGTexture, uint32>;
using FRDGBufferHandle        = TRDGHandle<FRDGBuffer, uint32>;
using FRDGViewHandle          = TRDGHandle<FRDGView, uint32>;
using FRDGUniformBufferHandle = TRDGHandle<FRDGUniformBuffer, uint32>;
using FRDGPassHandle          = TRDGHandle<FRDGPass, uint32>;

// 型付き Uniform Buffer
template <typename TUniformStruct>
using TRDGUniformBufferRef = TRDGUniformBuffer<TUniformStruct>*;
```

---

## リソースライフタイムフロー

```
CreateTexture("MyTex")           ← CPU メモリ確保のみ（GPU リソースなし）
  │
  └─ グラフに格納（Textures レジストリ）

AddPass([PassParams→OutUAV])      ← テクスチャへの参照を宣言（参照カウント加算）

Execute() 内
  ├─ Compile()                    ← 参照カウント = 0 ならカリング
  ├─ AllocatePooledTextures()     ← 最初の参照パスの直前に GPU メモリを確保
  ├─ パスごとの ExecutePassPrologue() ← バリア発行 + RenderPass 開始
  ├─ [ラムダ実行]  GetRHI() が有効になる
  └─ 最後の参照パス後             ← GPU メモリを解放（トランジェントなら即再利用）

QueueTextureExtraction 済みの場合:
  └─ Execute() 後に OutPooledTexture が有効化（次フレームへ持ち越し可能）
```

---

## FRDGResource

> **概要**: 全 RDG リソースの基底クラス。名前と RHI リソースポインタを管理する。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Name` | `const TCHAR* const` | デバッグ用リソース名（GPU ツールで表示される） |
| `ResourceRHI` | `FRHIResource*` | 実際の RHI リソース（`Execute()` 中のみ有効） |
| `bAllowRHIAccess` | `uint8` | `RDG_ENABLE_DEBUG` 時のみ：RHI アクセスが許可されているか |

### 主要関数

| 関数 | 説明 |
|-----|------|
| `GetRHI()` | パス内でのみ呼べる。デバッグビルドでアクセス時期を検証する |
| `MarkResourceAsUsed()` | 外部リソースを「実際に使用された」とマーク（カリング防止） |

---

## FRDGViewableResource

> **概要**: GPU メモリライフタイムを持つリソース（テクスチャ・バッファ）の基底クラス。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Type` | `ERDGViewableResourceType` | `Texture` or `Buffer` |
| `bExternal` | `uint8:1` | `RegisterExternalTexture` で登録されたか |
| `bExtracted` | `uint8:1` | `QueueTextureExtraction` が呼ばれたか |
| `bProduced` | `uint8:1` | 書き込みパスが存在するか（任意サブリソースへの書き込みで true） |
| `bTransient` | `uint8:1` | トランジェントアロケータ経由で確保されたか |
| `ReferenceCount` | `uint32` | 参照パス数（0 = カリング対象） |
| `EpilogueAccess` | `ERHIAccess` | グラフ終了時の最終アクセス状態（デフォルト `SRVMask`） |
| `AcquirePass` | `FRDGPassHandle` | GPU メモリを取得するパス |
| `DiscardPass` | `FRDGPassHandle` | GPU メモリを解放するパス |
| `FirstPass` | `FRDGPassHandle` | このリソースを最初に参照するパス |

### 主要関数

| 関数 | 戻り値 | 説明 |
|-----|--------|------|
| `IsExternal()` | `bool` | 外部登録リソースか |
| `IsExtracted()` | `bool` | 抽出キューに入っているか |
| `HasBeenProduced()` | `bool` | 書き込みパスが存在するか |
| `IsCullRoot()` | `bool` | External または Extracted → カリングルートになる（private） |

### 使用箇所
- [[ref_rdg_utils]] の `HasBeenProduced()` / `GetIfProduced()` ユーティリティ関数
- [[ref_rdg_builder]] の `UseExternalAccessMode()` でアクセスモードを切り替え

---

## FRDGTexture

> **概要**: RDG 管理テクスチャ。`FRDGBuilder::CreateTexture()` または `RegisterExternalTexture()` で生成。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Desc` | `const FRDGTextureDesc` | 解像度・フォーマット・ミップ数等の記述子 |
| `Flags` | `const ERDGTextureFlags` | `MultiFrame` / `SkipTracking` 等のフラグ |
| `Handle` | `FRDGTextureHandle` | グラフ内インデックス |
| `Layout` | `FRDGTextureSubresourceLayout` | サブリソース（Mip × Array × Plane）の構造 |
| `State` | `FRDGTextureSubresourceState` | グラフ構築中のサブリソースごとのアクセス状態 |
| `RenderTarget` | `IPooledRenderTarget*` | 割り当て済みプールテクスチャ（実行後に有効） |
| `TransientTexture` | `FRHITransientTexture*` | トランジェント割り当て（プラットフォーム対応時のみ） |
| `ViewCache` | `FRHITextureViewCache*` | SRV/UAV のキャッシュ（プール/トランジェントから取得） |
| `Allocation` | `TRefCountPtr<IPooledRenderTarget>` | 抽出時の強参照 |

### 主要関数

| 関数 | 説明 |
|-----|------|
| `GetRHI()` | `FRHITexture*` を返す（パス内でのみ有効） |
| `GetHandle()` | `FRDGTextureHandle` を返す |
| `GetSubresourceLayout()` | サブリソース構造（Mip/Array/Plane 数） |
| `GetSubresourceRange()` | 全サブリソース範囲 |
| `IsCulled()` | 参照カウントが 0 か |

### FRDGTextureDesc ファクトリ

```cpp
// 2D テクスチャ（最も一般的）
FRDGTextureDesc::Create2D(
    FIntPoint Size,
    EPixelFormat Format,
    FClearValueBinding ClearValue,
    ETextureCreateFlags Flags,
    uint8 NumMips = 1,
    uint8 NumSamples = 1);

// 2D テクスチャ配列
FRDGTextureDesc::Create2DArray(FIntPoint Size, EPixelFormat, FClearValueBinding, ETextureCreateFlags, uint16 ArraySize);

// 3D テクスチャ（ボリューメトリック）
FRDGTextureDesc::Create3D(FIntVector Size, EPixelFormat, FClearValueBinding, ETextureCreateFlags, uint8 NumMips = 1);

// Cube マップ
FRDGTextureDesc::CreateCube(uint32 Size, EPixelFormat, FClearValueBinding, ETextureCreateFlags, uint8 NumMips = 1);
```

### ERDGTextureFlags

| フラグ | 説明 |
|-------|------|
| `MultiFrame` | フレームをまたいで保持（RegisterExternal と組み合わせる） |
| `SkipTracking` | バリアトラッキングを除外（読み取り専用の外部リソースに使用） |
| `ForceImmediateFirstBarrier` | 最初のバリアを即時発行（スプリットバリアを禁止） |
| `MaintainCompression` | DCC メタデータを維持して帯域を節約 |

### 使用箇所
- あらゆるレンダリングターゲット・シェーダーリソースの宣言

---

## FRDGBuffer

> **概要**: RDG 管理バッファ。`FRDGBuilder::CreateBuffer()` または `RegisterExternalBuffer()` で生成。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Desc` | `FRDGBufferDesc` | バッファ記述子（要素サイズ・要素数・用途フラグ） |
| `Flags` | `const ERDGBufferFlags` | MultiFrame 等のフラグ |
| `Handle` | `FRDGBufferHandle` | グラフ内インデックス |
| `PooledBuffer` | `FRDGPooledBuffer*` | 割り当て済みプールバッファ |
| `TransientBuffer` | `FRHITransientBuffer*` | トランジェント割り当て |
| `NumElementsCallback` | `FRDGBufferNumElementsCallback*` | 要素数コールバック（後決め確保用） |
| `PendingCommitSize` | `uint64` | Reserved バッファの最初のトランジション時にコミットするサイズ |
| `State` | `FRDGSubresourceState*` | 現在のアクセス状態 |

### 主要関数

| 関数 | 説明 |
|-----|------|
| `GetRHI()` | `FRHIBuffer*` を返す（パス内でのみ有効） |
| `GetIndirectRHICallBuffer()` | DrawIndirect 用バッファとして返す（`BUF_DrawIndirect` フラグ必須） |
| `GetSize()` | バイト単位のバッファサイズ（`Desc.GetSize()`） |
| `GetStride()` | 1要素のバイト数（`Desc.BytesPerElement`） |
| `IsCulled()` | 参照カウントが 0 かつ PendingCommitSize が 0 か |

### FRDGBufferDesc ファクトリ

| ファクトリ | 用途 | HLSL 型 |
|-----------|------|---------|
| `CreateStructuredDesc(BytesPerElement, NumElements)` | 構造体バッファ | `StructuredBuffer<T>` |
| `CreateByteAddressDesc(NumBytes)` | バイトアドレスバッファ | `ByteAddressBuffer` |
| `CreateBufferDesc(BytesPerElement, NumElements)` | フォーマット付きバッファ | `Buffer<float4>` |
| `CreateIndirectDesc<TCmd>(NumCommands)` | 間接コマンドバッファ | `DrawIndirect` 等 |
| `CreateUploadDesc(BytesPerElement, NumElements)` | CPU→GPU 転送用 | — |

### ERDGBufferFlags

| フラグ | 説明 |
|-------|------|
| `MultiFrame` | フレームをまたいで保持 |
| `SkipTracking` | バリアトラッキングを除外（読み取り専用） |
| `ForceImmediateFirstBarrier` | 最初のバリアを即時発行 |

### 使用箇所
- Compute シェーダーのカウンタ・インダイレクトバッファ
- [[ref_rdg_allocator]] の `FRDGPooledBuffer` にプールされる

---

## FRDGView（SRV / UAV の基底）

> **概要**: テクスチャ / バッファの特定範囲にアクセスするためのビュークラスの基底。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Type` | `ERDGViewType` | TextureUAV / TextureSRV / BufferUAV / BufferSRV |
| `Handle` | `FRDGViewHandle` | グラフ内インデックス |
| `LastPass` | `FRDGPassHandle` | このビューを最後に参照するパス |

### ERDGViewType

```cpp
enum class ERDGViewType : uint8 {
    TextureUAV = 0,
    TextureSRV = 1,
    BufferUAV  = 2,
    BufferSRV  = 3,
    MAX
};
```

---

## FRDGUniformBuffer / TRDGUniformBuffer\<T\>

> **概要**: RDG グラフのライフタイムに紐付いた型付き Uniform Buffer。  
> `FRDGBuilder::CreateUniformBuffer()` で生成する。

### メンバ変数（FRDGUniformBuffer）

| 変数 | 型 | 説明 |
|-----|-----|------|
| `ParameterStruct` | `const FRDGParameterStruct` | パラメータ構造体のメタデータラッパー |
| `UniformBufferRHI` | `TRefCountPtr<FRHIUniformBuffer>` | RHI Uniform Buffer の参照 |
| `Handle` | `FRDGUniformBufferHandle` | グラフ内インデックス |
| `bExternal` | `bool` | 外部登録か否か |

### TRDGUniformBuffer\<T\> の追加 API

```cpp
const ParameterStructType* GetContents() const;     // C++ パラメータ構造体へのポインタ
TUniformBufferRef<T>        GetRHIRef() const;       // RHI Uniform Buffer の型付き参照
const ParameterStructType*  operator->() const;     // メンバへの直接アクセス
```

### 使用箇所
- `FSceneTextureUniformParameters` など、複数パスで共有する定数バッファ

---

## FRDGSubresourceState（内部バリア状態）

> **概要**: コンパイル中に各サブリソースの RHI アクセス状態を追跡する内部構造体。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Access` | `ERHIAccess` | 現在のアクセス状態（`SRVGraphics` / `UAVCompute` 等） |
| `FirstPass` | `FRDGPassHandlesByPipeline` | このアクセス状態を最初に使うパス（Graphics / AsyncCompute 別） |
| `LastPass` | `FRDGPassHandlesByPipeline` | このアクセス状態を最後に使うパス |
| `BarrierLocation` | `ERDGBarrierLocation` | バリアをプロローグ（前）かエピローグ（後）に置くか |
| `Flags` | `EResourceTransitionFlags` | バリアの追加フラグ |

### 使用箇所
- [[ref_rdg_builder]] の `Compile()` / `CollectPassBarriers()` 内部
