# リファレンス：RenderGraphUtils.h / RenderGraphUtils.cpp

- グループ: d - Parameters
- 上位: [[d_rdg_parameters]]
- 関連: [[ref_rdg_parameter]] | [[ref_rdg_event]] | [[ref_rdg_builder]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphUtils.h`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphUtils.cpp`

## 概要

レンダリングコードでよく使われる RDG ユーティリティ関数群。  
リソース状態確認・テクスチャ登録・クリア・コピー・リードバック・コンピュートパスのショートカットを含む。

---

## リソース状態確認ユーティリティ

```cpp
// テクスチャ / バッファが先行パスで生成済みか確認
bool HasBeenProduced(FRDGViewableResource* Resource);

// 生成済みならそのまま返し、未生成なら FallbackTexture を返す
FRDGTextureRef GetIfProduced(FRDGTextureRef Texture, FRDGTextureRef FallbackTexture = nullptr);
FRDGBufferRef  GetIfProduced(FRDGBufferRef Buffer,   FRDGBufferRef FallbackBuffer   = nullptr);

// 生成済みなら ELoad、未生成なら指定の ActionIfNotProduced を返す
ERenderTargetLoadAction GetLoadActionIfProduced(
    FRDGTextureRef Texture,
    ERenderTargetLoadAction ActionIfNotProduced);

// 生成済み判定に基づくバインディング生成
FRenderTargetBinding GetLoadBindingIfProduced(
    FRDGTextureRef Texture,
    ERenderTargetLoadAction ActionIfNotProduced);
```

### 使用箇所
- ポストプロセスパスで SceneColor が前段パスで書かれているか確認

---

## RHI リソース取得ユーティリティ

```cpp
// null チェック付きで RHI リソースを取得（nullptr 安全）
FRHITexture* TryGetRHI(FRDGTextureRef Texture);
FRHIBuffer*  TryGetRHI(FRDGBuffer*       Buffer);
FRHIBuffer*  TryGetRHI(FRDGPooledBuffer* Buffer);
FRHIShaderResourceView* TryGetSRV(FRDGPooledBuffer* Buffer);
uint64 TryGetSize(const FRDGBuffer*      Buffer);
uint64 TryGetSize(const FRDGPooledBuffer* Buffer);

// GraphBuilder にすでに登録されているか確認
bool IsRegistered(FRDGBuilder& GraphBuilder, const TRefCountPtr<IPooledRenderTarget>& RenderTarget);
bool IsRegistered(FRDGBuilder& GraphBuilder, const TRefCountPtr<FRDGPooledBuffer>& Buffer);
```

---

## テクスチャ登録ユーティリティ

```cpp
// null の場合に FallbackPooledTexture を使う安全版
FRDGTextureRef RegisterExternalTextureWithFallback(
    FRDGBuilder& GraphBuilder,
    const TRefCountPtr<IPooledRenderTarget>& ExternalPooledTexture,
    const TRefCountPtr<IPooledRenderTarget>& FallbackPooledTexture);

// null の場合に nullptr を返す（アサートしない）
FRDGTextureRef TryRegisterExternalTexture(
    FRDGBuilder& GraphBuilder,
    const TRefCountPtr<IPooledRenderTarget>& ExternalPooledTexture,
    ERDGTextureFlags Flags = ERDGTextureFlags::None);

FRDGBufferRef TryRegisterExternalBuffer(
    FRDGBuilder& GraphBuilder,
    const TRefCountPtr<FRDGPooledBuffer>& ExternalPooledBuffer,
    ERDGBufferFlags Flags = ERDGBufferFlags::None);

// FRHITexture* から直接登録（登録済みの場合は既存を返す）
FRDGTextureRef RegisterExternalTexture(
    FRDGBuilder& GraphBuilder,
    FRHITexture* Texture,
    const TCHAR* NameIfUnregistered,
    ERDGTextureFlags Flags = ERDGTextureFlags::None);
```

---

## レンダーターゲットバインディング生成

```cpp
// 複数テクスチャを同じ LoadAction で一括バインド
FRenderTargetBindingSlots GetRenderTargetBindings(
    ERenderTargetLoadAction ColorLoadAction,
    TArrayView<FRDGTextureRef> ColorTextures);

// FTextureRenderTargetBinding（bNeverClear / ArraySlice 対応）版
FRenderTargetBindingSlots GetRenderTargetBindings(
    ERenderTargetLoadAction ColorLoadAction,
    TArrayView<FTextureRenderTargetBinding> ColorTextures);
```

### FTextureRenderTargetBinding

```cpp
struct FTextureRenderTargetBinding
{
    FRDGTextureRef Texture    = nullptr;
    int16          ArraySlice = -1;       // -1 = 全スライス
    bool           bNeverClear = false;   // Clear 指定でも EClear を ELoad に差し替える
};
```

---

## MSAA ユーティリティ

```cpp
// MSAA テクスチャ対（Target + Resolve）を生成
FRDGTextureMSAA CreateTextureMSAA(
    FRDGBuilder& GraphBuilder,
    FRDGTextureDesc Desc,
    const TCHAR* NameMultisampled,
    const TCHAR* NameResolved,
    ETextureCreateFlags ResolveFlagsToAdd = TexCreate_None);

struct FRDGTextureMSAA {
    FRDGTextureRef Target  = nullptr;  // マルチサンプルテクスチャ
    FRDGTextureRef Resolve = nullptr;  // リゾルブ先（単サンプル）
    bool IsValid()    const;  // 両方非 null
    bool IsSeparate() const;  // Target != Resolve
};
```

---

## シェーダーパラメータ検証・クリア

```cpp
// シェーダーが使わないパラメータを nullptr / 0 にクリア
// （バリデーション失敗を事前に防ぐためにシェーダー設定前に呼ぶ）
template <typename TShaderClass>
void ClearUnusedGraphResources(
    const TShaderRef<TShaderClass>& Shader,
    typename TShaderClass::FParameters* InoutParameters,
    std::initializer_list<FRDGResourceRef> ExcludeList = {});

// 2 シェーダーの合成パラメータをクリア（VS + PS 等）
template <typename TShaderClassA, typename TShaderClassB, typename TPassParameterStruct>
void ClearUnusedGraphResources(
    const TShaderRef<TShaderClassA>& ShaderA,
    const TShaderRef<TShaderClassB>& ShaderB,
    TPassParameterStruct* InoutParameters,
    std::initializer_list<FRDGResourceRef> ExcludeList = {});
```

### 使用箇所
- `FComputeShaderUtils::AddPass()` 内部で自動的に呼び出す

---

## FComputeShaderUtils — Compute パスのショートカット

### GetGroupCount / GetGroupCountWrapped

```cpp
namespace FComputeShaderUtils {

// 定数: GCN の 1 wave / Nvidia の 2 warp を占有する推奨サイズ
static constexpr int32 kGolden2DGroupSize = 8;

// グループ数計算（スレッド数 ÷ グループサイズ、端数切り上げ）
FIntVector GetGroupCount(int32 ThreadCount, int32 GroupSize);
FIntVector GetGroupCount(FIntPoint ThreadCount, FIntPoint GroupSize);
FIntVector GetGroupCount(FIntPoint ThreadCount, int32 GroupSize);
FIntVector GetGroupCount(FIntVector ThreadCount, FIntVector GroupSize);

// X 次元がハードウェア上限を超える場合に Y/Z に折り返す
// シェーダー側で: uint LinearGroupId = GroupId.X + (GroupId.Z * 128 + GroupId.Y) * 128;
static constexpr int32 WrappedGroupStride = 128;
FIntVector GetGroupCountWrapped(int32 TargetGroupCount);
FIntVector GetGroupCountWrapped(int32 ThreadCount, int32 GroupSize);

} // namespace FComputeShaderUtils
```

### Dispatch / DispatchIndirect（直接 RHI 発行）

```cpp
// パイプライン設定 → パラメータ設定 → Dispatch → UAV アンセット
template<typename TShaderClass>
void Dispatch(
    FRHIComputeCommandList& RHICmdList,
    const TShaderRef<TShaderClass>& ComputeShader,
    const typename TShaderClass::FParameters& Parameters,
    FIntVector GroupCount);

// Indirect Dispatch（IndirectArgsBuffer のバリデーション付き）
template<typename TShaderClass>
void DispatchIndirect(
    FRHIComputeCommandList& RHICmdList,
    const TShaderRef<TShaderClass>& ComputeShader,
    const typename TShaderClass::FParameters& Parameters,
    FRDGBufferRef IndirectArgsBuffer,
    uint32 IndirectArgOffset);
```

### AddPass（RDG グラフへの追加）

```cpp
// グループ数固定版（内部で ClearUnusedGraphResources を自動実行）
template<typename TShaderClass>
FRDGPassRef AddPass(
    FRDGBuilder& GraphBuilder,
    FRDGEventName&& PassName,
    ERDGPassFlags PassFlags,          // Compute または AsyncCompute のみ有効
    const TShaderRef<TShaderClass>& ComputeShader,
    typename TShaderClass::FParameters* Parameters,
    FIntVector GroupCount);

// グループ数コールバック版（実行直前まで GroupCount を遅延決定できる）
// FRDGDispatchGroupCountCallback = TFunction<FIntVector()>
template<typename TShaderClass>
FRDGPassRef AddPass(
    FRDGBuilder& GraphBuilder,
    FRDGEventName&& PassName,
    ERDGPassFlags PassFlags,
    const TShaderRef<TShaderClass>& ComputeShader,
    typename TShaderClass::FParameters* Parameters,
    FRDGDispatchGroupCountCallback&& GroupCountCallback);

// ERDGPassFlags::Compute をデフォルト使用する簡略版
template<typename TShaderClass>
FRDGPassRef AddPass(
    FRDGBuilder& GraphBuilder,
    FRDGEventName&& PassName,
    const TShaderRef<TShaderClass>& ComputeShader,
    typename TShaderClass::FParameters* Parameters,
    FIntVector GroupCount);
```

### 内部処理フロー（AddPass）

```
AddPass(GraphBuilder, Name, PassFlags, ComputeShader, Parameters, GroupCount)
  ├─ checkf: PassFlags は Compute / AsyncCompute のみ許可
  ├─ ValidateGroupCount(GroupCount)
  ├─ ClearUnusedGraphResources(ComputeShader, Parameters)
  └─ GraphBuilder.AddPass(Name, Parameters, PassFlags,
         [ParametersMetadata, Parameters, ComputeShader, GroupCount](FRDGAsyncTask, FRHIComputeCommandList& RHICmdList)
         {
             FComputeShaderUtils::Dispatch(RHICmdList, ComputeShader, ParametersMetadata, *Parameters, GroupCount);
         })
```

---

## シェーダーパラメータマクロ（参照）

| マクロ | C++ 型 | 用途 |
|--------|--------|------|
| `SHADER_PARAMETER(Type, Name)` | プリミティブ | `float`, `uint32`, `FVector2f` 等 |
| `SHADER_PARAMETER_RDG_TEXTURE(T, N)` | `FRDGTextureRef` | SRV として使用 |
| `SHADER_PARAMETER_RDG_TEXTURE_SRV(T, N)` | `FRDGTextureSRVRef` | 明示的 SRV |
| `SHADER_PARAMETER_RDG_TEXTURE_UAV(T, N)` | `FRDGTextureUAVRef` | UAV |
| `SHADER_PARAMETER_RDG_BUFFER_SRV(T, N)` | `FRDGBufferSRVRef` | バッファ SRV |
| `SHADER_PARAMETER_RDG_BUFFER_UAV(T, N)` | `FRDGBufferUAVRef` | バッファ UAV |
| `SHADER_PARAMETER_RDG_UNIFORM_BUFFER(T, N)` | `TRDGUniformBufferRef<T>` | RDG UB |
| `RENDER_TARGET_BINDING_SLOTS()` | `FRenderTargetBindingSlots` | Raster パスのみ |

---

> [!note]- FRDGDispatchGroupCountCallback — 遅延 GroupCount
> ```cpp
> using FRDGDispatchGroupCountCallback = TFunction<FIntVector()>;
> ```
> GPU カウンタバッファの結果など、AddPass 時点では未確定だが  
> Execute 直前には確定している GroupCount を渡すための仕組み。  
> コールバックが返す GroupCount のいずれかが 0 の場合は Dispatch をスキップする。

> [!note]- GetGroupCountWrapped の注意点
> GLES 3.1 では X 次元の最大グループ数が 65535 と制限される場合がある。  
> `GetGroupCountWrapped` は `WrappedGroupStride = 128` でラップするため、  
> シェーダー側でも `GetUnWrappedDispatchGroupId(GroupId)` を使って線形 GroupId を復元する必要がある。

> [!note]- ValidateIndirectArgsBuffer のバリデーション内容
> 1. バッファに `BUF_VertexBuffer` or `BUF_ByteAddressBuffer` フラグが必要
> 2. バッファに `BUF_DrawIndirect` フラグが必要
> 3. IndirectArgOffset は 4 バイトアライン
> 4. プラットフォームによっては境界をまたぐオフセットが禁止（`PLATFORM_DISPATCH_INDIRECT_ARGUMENT_BOUNDARY_SIZE`）
