# リファレンス：FSubstrateGlobalUniformParameters / シェーダーバインド（Substrate.h）

- グループ: a / c - シェーダーパラメータ・バインド
- 上位: [[a_substrate_material]] | [[c_substrate_lighting]]
- 関連: [[ref_substrate_data]] | [[ref_substrate_classify]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Substrate/Substrate.h`

## 概要

Substrate の GPU シェーダーパラメータ構造体群と、各パスへのバインド関数をまとめたリファレンス。  
`FSubstrateGlobalUniformParameters` がすべてのシェーダーパスで共有されるメインの UBO。

---

## シェーダーパラメータ構造体

### `FSubstrateCommonParameters`（共通基底）

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FSubstrateCommonParameters, )
    SHADER_PARAMETER(uint32, MaxBytesPerPixel)      // ピクセルあたりの最大バイト数
    SHADER_PARAMETER(uint32, MaxClosurePerPixel)    // ピクセルあたりの最大クロージャ数
    SHADER_PARAMETER(uint32, bRoughDiffuse)         // Rough Diffuse 有効フラグ
    SHADER_PARAMETER(uint32, PeelLayersAboveDepth)  // デバッグ用レイヤーピール
    SHADER_PARAMETER(uint32, bRoughnessTracking)    // ラフネストラッキング有効フラグ
    SHADER_PARAMETER(uint32, bStochasticLighting)   // Stochastic Lighting 有効フラグ
END_SHADER_PARAMETER_STRUCT()
```

他の全構造体は `SHADER_PARAMETER_STRUCT_INCLUDE(FSubstrateCommonParameters, Common)` で包含する。

---

### `FSubstrateGlobalUniformParameters`（メイン UBO）

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FSubstrateGlobalUniformParameters, RENDERER_API)
    SHADER_PARAMETER_STRUCT_INCLUDE(FSubstrateCommonParameters, Common)
    SHADER_PARAMETER(int32,  SliceStoringDebugSubstrateTreeData) // デバッグスライスインデックス
    SHADER_PARAMETER(int32,  FirstSliceStoringSubstrateSSSData)  // SSS データ先頭スライス
    SHADER_PARAMETER(uint32, TileSize)                           // タイルサイズ（ピクセル）
    SHADER_PARAMETER(uint32, TileSizeLog2)                       // log2(TileSize)
    SHADER_PARAMETER(FIntPoint, TileCount)                       // タイルグリッドサイズ
    SHADER_PARAMETER_RDG_TEXTURE(Texture2DArray<uint>, MaterialTextureArray)   // クロージャデータ
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<SUBSTRATE_TOP_LAYER_TYPE>, TopLayerTexture)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float3>, OpaqueRoughRefractionTexture)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<uint>, ClosureOffsetTexture)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, ClosureTileBuffer)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, ClosureTileCountBuffer)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<uint4>, SampledMaterialTexture)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

シェーダーバインドポイント名: `"Substrate"`  
`IMPLEMENT_GLOBAL_SHADER_PARAMETER_STRUCT(FSubstrateGlobalUniformParameters, "Substrate")` で登録。

---

### `FSubstrateBasePassUniformParameters`（BasePass 用）

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FSubstrateBasePassUniformParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FSubstrateCommonParameters, Common)
    SHADER_PARAMETER(int32, SliceStoringDebugSubstrateTreeDataWithoutMRT) // MRT なしデバッグスライス
    SHADER_PARAMETER(int32, FirstSliceStoringSubstrateSSSDataWithoutMRT)  // MRT なし SSS スライス
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2DArray<uint>, MaterialTextureArrayUAVWithoutRTs) // 書き込み用 UAV
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float3>, OpaqueRoughRefractionTextureUAV)
END_SHADER_PARAMETER_STRUCT()
```

### `FSubstrateForwardPassUniformParameters`（Forward パス用）

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FSubstrateForwardPassUniformParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FSubstrateCommonParameters, Common)
    SHADER_PARAMETER(int32, FirstSliceStoringSubstrateSSSData)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2DArray<uint>, MaterialTextureArray)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<SUBSTRATE_TOP_LAYER_TYPE>, TopLayerTexture)
END_SHADER_PARAMETER_STRUCT()
```

### `FSubstratePublicParameters` / `FSubstratePublicGlobalUniformParameters`

外部モジュール向けの公開インターフェース（プラグイン等から参照可能）：

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FSubstratePublicParameters, RENDERER_API)
    SHADER_PARAMETER_STRUCT_INCLUDE(FSubstrateCommonParameters, Common)
    SHADER_PARAMETER(int32, FirstSliceStoringSubstrateSSSData)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<SUBSTRATE_TOP_LAYER_TYPE>, TopLayerTexture)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2DArray<uint>, MaterialTextureArray)
END_SHADER_PARAMETER_STRUCT()
```

### `FSubstrateTileParameter`（タイルパス用）

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FSubstrateTileParameter, )
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, TileListBuffer)
    SHADER_PARAMETER(uint32, TileListBufferOffset)
    SHADER_PARAMETER(uint32, TileEncoding)
    RDG_BUFFER_ACCESS(TileIndirectBuffer, ERHIAccess::IndirectArgs)
END_SHADER_PARAMETER_STRUCT()
```

---

## バインド関数一覧

| 関数 | バインド先 | 説明 |
|------|-----------|------|
| `Substrate::BindSubstrateBasePassUniformParameters()` | `FSubstrateBasePassUniformParameters` | BasePass バインド |
| `Substrate::BindSubstrateForwardPasslUniformParameters()` | `FSubstrateForwardPassUniformParameters` | Forward パスバインド |
| `Substrate::BindSubstrateMobileForwardPasslUniformParameters()` | Mobile Forward 用 | Mobile 対応 |
| `Substrate::BindSubstrateGlobalUniformParameters()` | `FSubstrateGlobalUniformParameters` | グローバル UBO バインド |
| `Substrate::BindSubstratePublicGlobalUniformParameters()` | `FSubstratePublicParameters` | 公開 UBO バインド |
| `Substrate::CreatePublicGlobalUniformBuffer()` | — | 公開グローバル UBO 生成 |
| `Substrate::SetTileParameters()` | `FSubstrateTileParameter` | タイルパスパラメータ取得 |

---

## `AppendSubstrateMRTs`

```cpp
void Substrate::AppendSubstrateMRTs(
    const FSceneRenderer& SceneRenderer,
    uint32& BasePassTextureCount,
    TArrayView<FTextureRenderTargetBinding> BasePassTextures);
```

BasePass の MRT スロットに Substrate テクスチャを追加する。  
`UsesMRT` モードのとき `MaterialTextureArray` の各スライスを MRT として追加。

---

## `SetBasePassRenderTargetOutputFormat`

```cpp
void Substrate::SetBasePassRenderTargetOutputFormat(
    const EShaderPlatform Platform,
    const FMaterialShaderParameters& MaterialParameters,
    FShaderCompilerEnvironment& OutEnvironment,
    EGBufferLayout GBufferLayout);
```

BasePass シェーダーコンパイル時に Substrate 専用の出力フォーマット定義をシェーダーに注入する。

---

## 使用箇所

- [[ref_substrate_data]] — `FSubstrateSceneData` のリソースをここに設定
- `FDeferredShadingSceneRenderer` — `BindSubstrateGlobalUniformParameters()` で各パスにバインド
- BasePass マテリアルシェーダー — `"Substrate"` ネームスペース経由でパラメータを読み取る
