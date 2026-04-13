# リファレンス：分類パス関数群 / FSubstrateTilePassVS（Substrate.h）

- グループ: b / c - タイル分類・ライティング
- 上位: [[b_substrate_classify]] | [[c_substrate_lighting]]
- 関連: [[ref_substrate_data]] | [[ref_substrate_params]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Substrate/Substrate.h/.cpp`

## 概要

Substrate タイル分類パスと、ライティングパスで使うタイル単位の Indirect Draw/Dispatch に関する  
関数群・シェーダークラスのリファレンス。

---

## ステンシルビット定数

```cpp
namespace Substrate
{
    constexpr uint32 StencilBit_Fast           = 0x10; // Fast パス（単純マテリアル）
    constexpr uint32 StencilBit_Single         = 0x20; // Single パス（中程度）
    constexpr uint32 StencilBit_Complex        = 0x40; // Complex パス（複数クロージャ）
    constexpr uint32 StencilBit_ComplexSpecial = 0x80; // Complex Special（Eye/Hair 等）
}
```

`SceneRenderTargets.h` の `STENCIL_SUBSTRATE_*` マクロと同期が必要。

---

## 分類パス関数

### `AddSubstrateMaterialClassificationPass`

```cpp
void Substrate::AddSubstrateMaterialClassificationPass(
    FRDGBuilder& GraphBuilder,
    const FMinimalSceneTextures& SceneTextures,
    const FDBufferTextures& DBufferTextures,
    const TArray<FViewInfo>& Views);
```

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `SceneTextures` | `const FMinimalSceneTextures&` | DepthStencil を含むシーンテクスチャ |
| `DBufferTextures` | `const FDBufferTextures&` | DBuffer テクスチャ |
| `Views` | `const TArray<FViewInfo>&` | 全ビュー |

処理: MaterialTextureArray を読み取り、ステンシルにタイル種別ビットを書き込み、タイルリストを生成。

### `AddSubstrateMaterialClassificationIndirectArgsPass`

```cpp
void Substrate::AddSubstrateMaterialClassificationIndirectArgsPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    ERDGPassFlags ComputePassFlags);
```

分類完了後、Indirect Draw/Dispatch の引数バッファを確定する。  
`r.Substrate.AsyncClassification=1` の場合は Shadow パスと並行実行。

### `AddSubstrateStencilPass`

```cpp
void Substrate::AddSubstrateStencilPass(
    FRDGBuilder& GraphBuilder,
    const TArray<FViewInfo>& Views,
    const FMinimalSceneTextures& SceneTextures);
```

ステンシルのみの書き込みパス。DBuffer 適用前に呼ばれることがある。

### `AddSubstrateDBufferPass`

```cpp
void Substrate::AddSubstrateDBufferPass(
    FRDGBuilder& GraphBuilder,
    const FSceneTextures& SceneTextures,
    const FDBufferTextures& DBufferTextures,
    const TArray<FViewInfo>& Views);
```

DBuffer デカールを Substrate マテリアルデータに適用するパス。

### `AddSubstrateSampleMaterialPass`

```cpp
void Substrate::AddSubstrateSampleMaterialPass(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FMinimalSceneTextures& SceneTextures,
    const TArray<FViewInfo>& Views);
```

Stochastic Lighting 用に MaterialTextureArray から代表値をサンプリングして  
`SampledMaterialTexture` に書き込む。

---

## `FSubstrateTilePassVS`

```cpp
class FSubstrateTilePassVS : public FGlobalShader
{
    DECLARE_GLOBAL_SHADER(FSubstrateTilePassVS);
    SHADER_USE_PARAMETER_STRUCT(FSubstrateTilePassVS, FGlobalShader);

    // パーミュテーション
    class FEnableDebug : SHADER_PERMUTATION_BOOL("PERMUTATION_ENABLE_DEBUG");
    class FEnableTexCoordScreenVector : SHADER_PERMUTATION_BOOL("PERMUTATION_ENABLE_TEXCOORD_SCREENVECTOR");
    using FPermutationDomain = TShaderPermutationDomain<FEnableDebug, FEnableTexCoordScreenVector>;

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
        SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, TileListBuffer)
        SHADER_PARAMETER(uint32, TileListBufferOffset)
        SHADER_PARAMETER(uint32, TileEncoding)
        RDG_BUFFER_ACCESS(TileIndirectBuffer, ERHIAccess::IndirectArgs)
    END_SHADER_PARAMETER_STRUCT()
};
```

タイルリストバッファからタイル座標を取得し、それを頂点として展開する頂点シェーダー。  
ライティングパスでは `TileIndirectBuffer` を使って Indirect Draw を発行する。

### `static bool ShouldCompilePermutation(...)`

プラットフォームが Substrate をサポートしている場合にのみコンパイル。

### `static void ModifyCompilationEnvironment(...)`

`SUBSTRATE_TILETYPE` 等の定義をシェーダー環境に追加。

---

## タイルパラメータ取得関数

### `SetTileParameters`（オーバーロード1）

```cpp
FSubstrateTileParameter Substrate::SetTileParameters(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const ESubstrateTileType Type);
```

`FSubstrateTileParameter` を取得する。`FViewInfo::SubstrateViewData` から SRV と Indirect Args を解決する。

### `SetTileParameters`（オーバーロード2・3）

```cpp
FSubstrateTilePassVS::FParameters Substrate::SetTileParameters(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const ESubstrateTileType Type,
    EPrimitiveType& PrimitiveType);

FSubstrateTilePassVS::FParameters Substrate::SetTileParameters(
    const FViewInfo& View,
    const ESubstrateTileType Type,
    EPrimitiveType& PrimitiveType);
```

`FSubstrateTilePassVS::FParameters` として取得（vs に直接渡す形式）。  
`PrimitiveType` に描画プリミティブタイプ（通常 `PT_TriangleList`）が設定される。

---

## オフセット計算関数

### `TileTypeDrawIndirectArgOffset`

```cpp
uint32 Substrate::TileTypeDrawIndirectArgOffset(const ESubstrateTileType Type);
```

`ClassificationTileDrawIndirectBuffer` 内でのタイル種別ごとのバイトオフセットを返す。

### `TileTypeDispatchIndirectArgOffset`

```cpp
uint32 Substrate::TileTypeDispatchIndirectArgOffset(const ESubstrateTileType Type);
```

`ClassificationTileDispatchIndirectBuffer` 内でのオフセットを返す。

### `GetClosureTileIndirectArgsOffset`

```cpp
uint32 Substrate::GetClosureTileIndirectArgsOffset(uint32 InDownsampleFactor);
```

クロージャタイルの Dispatch Indirect Args のオフセットを返す（ダウンサンプル係数で変化）。

---

## ユーティリティ

### `IsStochasticLightingActive`

```cpp
bool Substrate::IsStochasticLightingActive(EShaderPlatform In);
```

プラットフォームと `r.Substrate.StochasticLighting.Active` を参照して判定。

### `ShouldRenderSubstrateDebugPasses`

```cpp
bool Substrate::ShouldRenderSubstrateDebugPasses(const FViewInfo& View);
```

デバッグパスを実行すべきか判定。`r.Substrate.Debug.*` CVar を参照する。

### `AddSubstrateDebugPasses`

```cpp
FScreenPassTexture Substrate::AddSubstrateDebugPasses(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture& ScreenPassSceneColor);
```

デバッグオーバーレイをシーンカラーに描画して返す。

---

## `FSubstrateTileType`（シェーダーパーミュテーション）

```cpp
class FSubstrateTileType : SHADER_PERMUTATION_INT("SUBSTRATE_TILETYPE", 4);
```

値 0〜3 が Fast / Single / Complex / ComplexSpecial に対応。  
`ShouldCompileSubstrateTileTypePermutations()` でプラットフォームごとのコンパイル可否を制御。

---

## 使用箇所

- [[ref_substrate_data]] — `FSubstrateViewData` のタイルバッファを SRV として参照
- [[ref_substrate_params]] — `FSubstrateTileParameter` はライティングパスのシェーダーに渡される
- `FDeferredShadingSceneRenderer` — `AddSubstrateMaterialClassificationPass()` を BasePass 後に呼び出す
