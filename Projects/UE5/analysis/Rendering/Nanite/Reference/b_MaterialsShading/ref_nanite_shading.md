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

## 主要関数

| 関数 | 説明 |
|------|------|
| `ShadeBinning(FRDGBuilder&, FRasterResults&, ...)` | VisBuffer のピクセルをマテリアルビンに分類 |
| `BuildShadingCommands(FRDGBuilder&, ...)` | ビン別の Indirect Dispatch 引数を生成 |
| `LoadBasePassPipeline(...)` | ベースパス用シェーディングパイプラインをロード |
| `LoadLumenCardPipeline(...)` | Lumen Surface Cache キャプチャ用パイプラインをロード |
| `DispatchBasePass(FRDGBuilder&, FShadeBinning&, ...)` | ビン単位でシェーダーをディスパッチ → GBuffer 書き込み |
| `DispatchLumenMeshCapturePass(FRDGBuilder&, ...)` | Lumen カード用キャプチャパスをディスパッチ |
