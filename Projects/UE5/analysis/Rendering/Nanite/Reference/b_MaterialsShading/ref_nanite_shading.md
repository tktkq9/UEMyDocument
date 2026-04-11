# リファレンス：NaniteShading.h / NaniteShading.cpp

- グループ: b - Materials & Shading
- 上位: [[b_nanite_materials_shading]]
- 関連: [[ref_nanite_materials]] | [[ref_nanite_draw_list]] | [[ref_nanite_cull_raster]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteShading.h/.cpp`

---

## 概要

VisBuffer から生成したマテリアルビンに対して、  
コンピュートシェーダーをディスパッチして GBuffer や Lumen Surface Cache に書き込むシステム。  
`ShadeBinning()` でピクセルをビンに分類し、  
`DispatchBasePass()` または `DispatchLumenMeshCapturePass()` で実際にシェーディングを実行する。

---

## 主要クラス・構造体

```cpp
namespace Nanite
{

// シェーディングビン情報（分類後のビン状態）
struct FShadeBinning
{
    FRDGBufferRef ShadingBinArgs;      // Indirect Dispatch 引数バッファ
    FRDGBufferRef ShadingBinData;      // ビン別データ（オフセット・フラグ）
    FRDGBufferRef ShadingBinStats;     // デバッグ統計
    uint32 ShadingBinCount;            // 有効なビン数
};

// シェーディングパイプライン（1マテリアル = 1パイプライン）
struct FNaniteShadingPipeline
{
    FRHIComputePipelineState* ComputePipelineState; // CS パイプラインオブジェクト
    TShaderRef<FNaniteMaterialShader> Shader;       // シェーダー参照
    bool bIsProgrammable;     // プログラマブルシェーダーか（マスク・WPO 等）
    bool bForceFullyRough;    // 完全ラフ強制（パフォーマンスモード）
};

} // namespace Nanite
```

---

## FShadeBinning — メンバ変数

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `ShadingBinArgs` | `FRDGBufferRef` | IndirectDispatch 引数バッファ（x/y/z グループ数）|
| `ShadingBinData` | `FRDGBufferRef` | ビン別データ（オフセット・フラグ） |
| `ShadingBinStats` | `FRDGBufferRef` | デバッグ統計（有効時のみ） |
| `FastClearVisualize` | `FRDGTextureRef` | ファストクリア可視化テクスチャ |
| `DataByteOffset` | `uint32` | ビンデータの開始オフセット |

---

## FNaniteShadingPipeline — メンバ変数

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `ComputePipelineState` | `FRHIComputePipelineState*` | CS パイプラインオブジェクト |
| `Shader` | `TShaderRef<FNaniteMaterialShader>` | シェーダー参照 |
| `bIsProgrammable` | `bool` | プログラマブルシェーダーか（マスク・WPO 等）|
| `bForceFullyRough` | `bool` | 完全ラフ強制（パフォーマンスモード）|

---

## 主要関数

| 関数 | ファイル:行 | 説明 |
|------|-----------|------|
| `ShadeBinning(FRDGBuilder&, FRasterResults&, ...)` | `NaniteShading.cpp:1443` | VisBuffer ピクセルをビンに分類（3 CS パス）|
| `BuildShadingCommands(FRDGBuilder&, ...)` | `NaniteShading.h:49` | ビン別 IndirectDispatch 引数構築 |
| `LoadBasePassPipeline(...)` | `NaniteShading.h:57` | BasePass 用シェーディングパイプラインロード |
| `LoadLumenCardPipeline(...)` | `NaniteShading.h:65` | Lumen Surface Cache 用パイプラインロード |
| `DispatchBasePass(FRDGBuilder&, FShadeBinning&, ...)` | `NaniteShading.cpp:1178` | ビン単位 CS ディスパッチ → GBuffer |
| `DispatchLumenMeshCapturePass(FRDGBuilder&, ...)` | `NaniteShading.cpp:2472` | Lumen カードキャプチャパスのディスパッチ |

---

> [!note]- ShadeBinning の3パスCS構成
> `ShadeBinning()` 内では以下の3つのコンピュートシェーダーを順次実行する:  
> 1. `ClassifyPixels CS` — VisBuffer64 の各ピクセルから MaterialDepth を読み取り、ビンインデックスに変換  
> 2. `ShadingBinBuildCS` — ビンごとのピクセル数をカウントして `ShadingBinData` を構築  
> 3. `ShadingBinReserveCS` — ビン別オフセットを確定し、`ShadingBinArgs` に IndirectDispatch x/y/z を書き込む  

> [!note]- BuildShadingCommands と DispatchBasePass の分離
> `BuildShadingCommands()` は RenderBasePass の前（`DeferredShadingRenderer.cpp:2599`）で呼ばれ、  
> `DispatchBasePass()` は RenderBasePass 内（`BasePassRendering.cpp:1543`）で呼ばれる。  
> この分離により、マテリアル変更がない場合に `BuildShadingCommands()` の結果をキャッシュして再利用できる。

> [!note]- bIsProgrammable のコスト影響
> `FNaniteShadingPipeline::bIsProgrammable = true`（WPO / マスク / PDO 等）の場合、  
> HW ラスタライザが使えず SW（Compute）ラスタに強制される。  
> また専用のシェーダーパイプラインが必要なため、プログラマブルマテリアルの比率が高いと  
> IndirectDispatch の回数が増加しパフォーマンスに影響する。
