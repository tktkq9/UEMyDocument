# リファレンス：RenderGraphParameter.h / RenderGraphParameters.inl

- グループ: d - Parameters
- 上位: [[d_rdg_parameters]]
- 関連: [[ref_rdg_blackboard]] | [[ref_rdg_utils]] | [[ref_rdg_builder]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphParameter.h`
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphParameters.inl`

## 概要

パス宣言時に渡すパラメータ構造体の内部表現 `FRDGParameter` と  
その集合ラッパー `FRDGParameterStruct` を定義する。  
RDG はこの型を使ってリソース依存グラフを自動構築する。

---

## FRDGParameter

パラメータ構造体内の 1 フィールドを表すクラス。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `MemberType` | `EUniformBufferBaseType` | フィールドの型識別子（`UBMT_xxx`） |
| `MemberPtr` | `void* const` | 構造体バイト列内のフィールドアドレス |

### UBMT_xxx 型一覧

| `UBMT_xxx` 値 | C++ 型 | 意味 |
|--------------|--------|------|
| `UBMT_RDG_TEXTURE` | `FRDGTextureRef` | RDG テクスチャ（直接） |
| `UBMT_RDG_TEXTURE_SRV` | `FRDGTextureSRVRef` | テクスチャ SRV（全シェーダーステージ） |
| `UBMT_RDG_TEXTURE_NON_PIXEL_SRV` | `FRDGTextureSRVRef` | テクスチャ SRV（非ピクセルシェーダー） |
| `UBMT_RDG_TEXTURE_UAV` | `FRDGTextureUAVRef` | テクスチャ UAV |
| `UBMT_RDG_TEXTURE_ACCESS` | `FRDGTextureAccess` | アクセス状態付きテクスチャ |
| `UBMT_RDG_TEXTURE_ACCESS_ARRAY` | `FRDGTextureAccessArray` | アクセス状態付きテクスチャの配列 |
| `UBMT_RDG_BUFFER_SRV` | `FRDGBufferSRVRef` | バッファ SRV |
| `UBMT_RDG_BUFFER_UAV` | `FRDGBufferUAVRef` | バッファ UAV |
| `UBMT_RDG_BUFFER_ACCESS` | `FRDGBufferAccess` | アクセス状態付きバッファ |
| `UBMT_RDG_BUFFER_ACCESS_ARRAY` | `FRDGBufferAccessArray` | アクセス状態付きバッファの配列 |
| `UBMT_RDG_UNIFORM_BUFFER` | `FRDGUniformBufferBinding` | RDG 管理の Uniform Buffer |
| `UBMT_RENDER_TARGET_BINDING_SLOTS` | `FRenderTargetBindingSlots` | レンダーターゲット |

### 判定メソッド

```cpp
bool IsResource()               const;  // RenderTargetBindingSlots と ResourceAccessArray 以外
bool IsSRV()                    const;  // TEXTURE_SRV, TEXTURE_NON_PIXEL_SRV, BUFFER_SRV
bool IsUAV()                    const;  // TEXTURE_UAV, BUFFER_UAV
bool IsView()                   const;  // IsSRV() || IsUAV()
bool IsTexture()                const;  // RDG_TEXTURE or RDG_TEXTURE_ACCESS
bool IsTextureAccess()          const;  // RDG_TEXTURE_ACCESS
bool IsTextureAccessArray()     const;  // RDG_TEXTURE_ACCESS_ARRAY
bool IsBuffer()                 const;  // RDG_BUFFER_ACCESS
bool IsBufferAccess()           const;  // RDG_BUFFER_ACCESS
bool IsBufferAccessArray()      const;  // RDG_BUFFER_ACCESS_ARRAY
bool IsResourceAccessArray()    const;  // IsBufferAccessArray() || IsTextureAccessArray()
bool IsUniformBuffer()          const;  // RDG_UNIFORM_BUFFER
bool IsViewableResource()       const;  // IsTexture() || IsBuffer()
bool IsRenderTargetBindingSlots() const;

// 型を指定して値を取得（型が一致しない場合は check() でアサート）
FRDGResourceRef                GetAsResource()             const;
FRDGUniformBufferBinding       GetAsUniformBuffer()        const;
FRDGViewableResource*          GetAsViewableResource()     const;
FRDGViewRef                    GetAsView()                 const;
FRDGShaderResourceViewRef      GetAsSRV()                  const;
FRDGUnorderedAccessViewRef     GetAsUAV()                  const;
FRDGTextureRef                 GetAsTexture()              const;
FRDGTextureAccess              GetAsTextureAccess()        const;
const FRDGTextureAccessArray&  GetAsTextureAccessArray()   const;
FRDGBufferRef                  GetAsBuffer()               const;
FRDGBufferAccess               GetAsBufferAccess()         const;
const FRDGBufferAccessArray&   GetAsBufferAccessArray()    const;
FRDGTextureSRVRef              GetAsTextureSRV()           const;
FRDGBufferSRVRef               GetAsBufferSRV()            const;
FRDGTextureUAVRef              GetAsTextureUAV()           const;
FRDGBufferUAVRef               GetAsBufferUAV()            const;
const FRenderTargetBindingSlots& GetAsRenderTargetBindingSlots() const;
```

### 使用箇所
- [[ref_rdg_builder]] `FRDGBuilder::SetupPassDependencies()` — `EnumerateTextures` / `EnumerateBuffers` でパス依存を自動追跡
- [[ref_rdg_validation]] `FRDGUserValidation` — パラメータの整合性チェック

---

## FRDGParameterStruct

パラメータ構造体全体のラッパー。  
`BEGIN_SHADER_PARAMETER_STRUCT` で宣言した構造体から自動生成される。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Contents` | `const uint8*` | 構造体の生バイト列先頭 |
| `Layout` | `FUniformBufferLayoutRHIRef` | フィールドのオフセット・型情報 |
| `Metadata` | `const FShaderParametersMetadata*` | HLSL シェーダーバインド情報（省略可能） |

### 主要メソッド

```cpp
// レイアウト情報
const FRHIUniformBufferLayout& GetLayout()    const;
bool HasRenderTargets()                        const;
bool HasExternalOutputs()                      const;
bool HasStaticSlot()                           const;

// パラメータ数
uint32 GetBufferParameterCount()               const;
uint32 GetTextureParameterCount()              const;
uint32 GetUniformBufferParameterCount()        const;

// 列挙（RDG 内部でリソース依存の解析に使用）
template <typename FunctionType>
void Enumerate(FunctionType Function)          const;  // 全パラメータ（Uniform Buffer も再帰的に含む）
void EnumerateTextures(FunctionType Function)  const;
void EnumerateBuffers(FunctionType Function)   const;
void EnumerateUniformBuffers(FunctionType Function) const;

// レンダーターゲット
const FRenderTargetBindingSlots& GetRenderTargets() const;  // HasRenderTargets() が true の場合のみ有効

// RHI バインド情報生成
FRHIRenderPassInfo GetRenderPassInfo() const;
FUniformBufferStaticBindings GetStaticUniformBuffers() const;

// クリーンアップ
static void ClearUniformBuffers(void* Contents, const FRHIUniformBufferLayout* Layout);
```

### 使用箇所
- [[ref_rdg_pass]] `FRDGPass::ParameterStruct` — パスが保持する参照
- [[ref_rdg_builder]] `FRDGBuilder::SetupPassDependencies()` — パス間依存グラフの自動構築

---

## TRDGParameterStruct（型付きラッパー）

`FRDGParameterStruct` を型付きで使うサブクラス。

```cpp
template <typename ParameterStructType>
class TRDGParameterStruct : public FRDGParameterStruct
{
public:
    explicit TRDGParameterStruct(ParameterStructType* Parameters);

    const ParameterStructType* GetContents() const;
    const ParameterStructType* operator->()  const;
};
```

---

## ヘルパー関数

```cpp
// SkipRenderPass フラグと組み合わせて手動でレンダーパスを制御する場合に使用
template <typename TParameterStruct>
FRHIRenderPassInfo GetRenderPassInfo(TParameterStruct* Parameters);

// パラメータ構造体にレンダーターゲットが含まれるか判定
template <typename TParameterStruct>
bool HasRenderPassInfo(TParameterStruct* Parameters);

// グローバル Uniform Buffer バインド情報を取得
template <typename TParameterStruct>
FUniformBufferStaticBindings GetStaticUniformBuffers(TParameterStruct* Parameters);
```

---

## シェーダーパラメータマクロ

```cpp
// ──────────────────── 構造体の宣言 ────────────────────
BEGIN_SHADER_PARAMETER_STRUCT(FMyPassParameters, )
    // テクスチャ（直接アクセス）
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, InTexture)
    SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture2D, InSRV)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float4>, OutUAV)
    SHADER_PARAMETER_RDG_TEXTURE_ARRAY(Texture2D, InArray, [4])

    // バッファ
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<FMyData>, InBuf)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<FMyData>, OutBuf)

    // Uniform Buffer（型付き）
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneTextureUniformParameters, SceneTextures)
    SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)

    // スカラー / ベクター
    SHADER_PARAMETER(float, MyFloat)
    SHADER_PARAMETER(FVector2f, UVScale)
    SHADER_PARAMETER(uint32, NumElements)

    // Raster パスのレンダーターゲット（最後に置く）
    RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()
```

### マクロ一覧

| マクロ | C++ 型 | HLSL 型 |
|--------|--------|---------|
| `SHADER_PARAMETER_RDG_TEXTURE` | `FRDGTextureRef` | `Texture2D` 等 |
| `SHADER_PARAMETER_RDG_TEXTURE_SRV` | `FRDGTextureSRVRef` | `Texture2D` |
| `SHADER_PARAMETER_RDG_TEXTURE_NON_PIXEL_SRV` | `FRDGTextureSRVRef` | 非ピクセルシェーダー専用 SRV |
| `SHADER_PARAMETER_RDG_TEXTURE_UAV` | `FRDGTextureUAVRef` | `RWTexture2D` |
| `SHADER_PARAMETER_RDG_TEXTURE_ARRAY` | `FRDGTextureRef[N]` | `Texture2D` 配列 |
| `SHADER_PARAMETER_RDG_BUFFER_SRV` | `FRDGBufferSRVRef` | `StructuredBuffer<>` |
| `SHADER_PARAMETER_RDG_BUFFER_UAV` | `FRDGBufferUAVRef` | `RWStructuredBuffer<>` |
| `SHADER_PARAMETER_RDG_UNIFORM_BUFFER` | `TRDGUniformBufferRef<T>` | `cbuffer` |
| `SHADER_PARAMETER_STRUCT_REF` | `TUniformBufferRef<T>` | `cbuffer` |
| `SHADER_PARAMETER` | プリミティブ型 | `float`, `uint` 等 |
| `RENDER_TARGET_BINDING_SLOTS` | `FRenderTargetBindingSlots` | — |

---

> [!note]- FRDGTextureAccess / FRDGBufferAccess（アクセス状態付きラッパー）
> ```cpp
> struct FRDGTextureAccess {
>     FRDGTextureRef Texture;
>     ERHIAccess     Access;   // パスで要求するアクセス状態
> };
> struct FRDGBufferAccess {
>     FRDGBufferRef Buffer;
>     ERHIAccess    Access;
> };
> ```
> `SHADER_PARAMETER_RDG_TEXTURE` / `BUFFER` の代わりに明示的にアクセス状態を指定したい場合に使う。  
> `UBMT_RDG_TEXTURE_ACCESS` / `UBMT_RDG_BUFFER_ACCESS` として RDG に認識される。

> [!note]- FRDGParameterStruct の Layout と Metadata の違い
> - `Layout`（`FRHIUniformBufferLayout`）: RHI レイヤが参照するバインド情報（オフセット・型・サイズ）。常に存在する。
> - `Metadata`（`FShaderParametersMetadata`）: HLSL シェーダーとの名前マッピング情報。デバッグ・リフレクション用で、コンストラクタに渡さない場合は nullptr。

> [!note]- Enumerate と EnumerateTextures / EnumerateBuffers の違い
> `Enumerate` は `UBMT_RDG_UNIFORM_BUFFER` の中身も **再帰的に展開** してすべてのパラメータを列挙する。  
> `EnumerateTextures` / `EnumerateBuffers` は再帰なし（トップレベルのみ）。  
> RDG 内部の依存解析は再帰展開が必要なため `Enumerate` を使用する。
