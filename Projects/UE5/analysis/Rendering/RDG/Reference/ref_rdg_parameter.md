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

```cpp
class FRDGParameter final
{
public:
    // フィールド種別の判定
    bool IsResource()           const;  // テクスチャ / バッファ SRV/UAV
    bool IsSRV()                const;  // SRV（Texture / Buffer）
    bool IsUAV()                const;  // UAV（Texture / Buffer）
    bool IsView()               const;  // SRV || UAV
    bool IsTexture()            const;  // RDG テクスチャ（SRV/UAV 以外）
    bool IsTextureAccess()      const;  // UBMT_RDG_TEXTURE_ACCESS
    bool IsTextureAccessArray() const;  // UBMT_RDG_TEXTURE_ACCESS_ARRAY
    bool IsBuffer()             const;  // RDG バッファアクセス
    bool IsBufferAccess()       const;  // UBMT_RDG_BUFFER_ACCESS
    bool IsRenderTargetBindingSlots() const;
    bool IsResourceAccessArray()      const;
    bool IsUniformBuffer()      const;  // Uniform Buffer（RDG 管理外含む）
    bool IsRDGResource()        const;  // RDG 管理リソースのみ true
};
```

### メンバ型 (UBMT_xxx)

フィールドの型はシェーダーパラメータメタデータ型 `EUniformBufferMemberType`（`UBMT_xxx`）で識別される。

| `UBMT_xxx` 値 | 意味 |
|--------------|------|
| `UBMT_RDG_TEXTURE` | `FRDGTextureRef`（直接テクスチャ） |
| `UBMT_RDG_TEXTURE_SRV` | `FRDGTextureSRVRef` |
| `UBMT_RDG_TEXTURE_UAV` | `FRDGTextureUAVRef` |
| `UBMT_RDG_BUFFER_SRV` | `FRDGBufferSRVRef` |
| `UBMT_RDG_BUFFER_UAV` | `FRDGBufferUAVRef` |
| `UBMT_RDG_TEXTURE_ACCESS` | `FRDGTextureAccess`（アクセス状態付き） |
| `UBMT_RDG_BUFFER_ACCESS` | `FRDGBufferAccess`（アクセス状態付き） |

---

## FRDGParameterStruct

パラメータ構造体全体のラッパー。  
`BEGIN_SHADER_PARAMETER_STRUCT` で宣言した構造体から自動生成される。

```cpp
class FRDGParameterStruct
{
public:
    // レンダーターゲット / 外部出力の有無
    bool HasRenderTargets()       const;
    bool HasExternalOutputs()     const;

    // パラメータ数
    uint32 GetBufferParameterCount()        const;
    uint32 GetTextureParameterCount()       const;
    uint32 GetUniformBufferParameterCount() const;

    // パラメータを列挙（RDG 内部でリソース依存の解析に使用）
    template <typename FunctionType>
    void EnumerateTextures(FunctionType Function)        const;
    void EnumerateBuffers(FunctionType Function)         const;
    void EnumerateUniformBuffers(FunctionType Function)  const;
};
```

---

## シェーダーパラメータマクロ（シェーダーバインドと連動）

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

---

## マクロ一覧

| マクロ | C++ 型 | HLSL 型 |
|--------|--------|---------|
| `SHADER_PARAMETER_RDG_TEXTURE` | `FRDGTextureRef` | `Texture2D` 等 |
| `SHADER_PARAMETER_RDG_TEXTURE_SRV` | `FRDGTextureSRVRef` | `Texture2D` |
| `SHADER_PARAMETER_RDG_TEXTURE_UAV` | `FRDGTextureUAVRef` | `RWTexture2D` |
| `SHADER_PARAMETER_RDG_TEXTURE_ARRAY` | `FRDGTextureRef[N]` | `Texture2D` 配列 |
| `SHADER_PARAMETER_RDG_BUFFER_SRV` | `FRDGBufferSRVRef` | `StructuredBuffer<>` |
| `SHADER_PARAMETER_RDG_BUFFER_UAV` | `FRDGBufferUAVRef` | `RWStructuredBuffer<>` |
| `SHADER_PARAMETER_RDG_UNIFORM_BUFFER` | `TRDGUniformBufferRef<T>` | `cbuffer` |
| `SHADER_PARAMETER_STRUCT_REF` | `TUniformBufferRef<T>` | `cbuffer` |
| `SHADER_PARAMETER` | プリミティブ型 | `float`, `uint` 等 |
| `RENDER_TARGET_BINDING_SLOTS` | `FRenderTargetBindingSlots` | — |
