# リファレンス：NaniteDrawList.h / NaniteDrawList.cpp

- グループ: b - Materials & Shading
- 上位: [[b_nanite_materials_shading]]
- 関連: [[ref_nanite_materials]] | [[ref_nanite_shading]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteDrawList.h/.cpp`

---

## 概要

マテリアル別のラスタライズビン・シェーディングビンを蓄積し、  
描画コマンドとして確定するドロー命令リスト管理モジュール。  
`FNaniteMaterialListContext` を通じてビンを登録し、  
`FDeferredPipelines` でパイプラインの遅延構築（ゲームスレッドからの並列ビルド）を管理する。

---

## 主要クラス

```cpp
namespace Nanite
{

// マテリアルリストコンテキスト（ビン登録インターフェース）
class FNaniteMaterialListContext
{
public:
    // ラスタライズビンを登録
    void AddRasterBin(
        const FNaniteRasterBin& Bin,
        const FMeshBatch& MeshBatch,
        const FMaterial& Material);

    // シェーディングビンを登録
    void AddShadingBin(
        const FNaniteShadingBin& Bin,
        const FMeshBatch& MeshBatch,
        const FMaterial& Material);

    // 登録内容を適用（ソート・重複排除）
    void Apply(FNaniteRasterPipelines& RasterPipelines,
               FNaniteShadingPipelines& ShadingPipelines);
};

// 遅延パイプライン情報（ゲームスレッドからビルドして RT で適用）
struct FDeferredPipelines
{
    TArray<FNaniteRasterPipeline> RasterPipelines;
    TArray<FNaniteShadingPipeline> ShadingPipelines;

    // 並列ビルドタスクが完了するまで待機
    void WaitForTaskCompletion();
};

} // namespace Nanite
```

---

## 主要関数

| 関数 | 説明 |
|------|------|
| `AddRasterBin(...)` | ラスタライズビンをリストに追加 |
| `AddShadingBin(...)` | シェーディングビンをリストに追加 |
| `Apply(...)` | ビンをソート・重複排除して確定パイプラインに変換 |
| `WaitForTaskCompletion()` | 並列ビルドタスクの完了待機 |
