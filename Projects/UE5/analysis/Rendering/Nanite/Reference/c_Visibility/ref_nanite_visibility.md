# リファレンス：NaniteVisibility.h / NaniteVisibility.cpp

- グループ: c - Visibility
- 上位: [[c_nanite_visibility]]
- 関連: [[ref_nanite_ownership_visibility]] | [[ref_nanite_cull_raster]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteVisibility.h/.cpp`

---

## 概要

Nanite カリングパスの結果から「どのラスタライズビン・シェーディングビンが可視か」を  
非同期並列タスクで判定するシステム。  
シェーディングビンの可視性によって不要なディスパッチをスキップし、GPU 負荷を削減する。

---

## 主要クラス・構造体

```cpp
namespace Nanite
{

// 可視性テスト結果（ビン単位のビットマスク）
struct FNaniteVisibilityResults
{
    // ラスタライズビンの可視ビットマスク（32ビット/エントリ）
    TArray<uint32> RasterBinVisibility;
    // シェーディングビンの可視ビットマスク
    TArray<uint32> ShadingBinVisibility;

    // 判定ヘルパー
    bool IsRasterBinVisible(uint16 BinIndex) const;
    bool IsShadingBinVisible(uint16 BinIndex) const;
    bool IsAnyRasterBinVisible() const;
    bool IsAnyShadingBinVisible() const;
    int32 GetNumVisibleRasterBins() const;
    int32 GetNumVisibleShadingBins() const;
};

// 可視性クエリ（非同期タスクのハンドル）
struct FNaniteVisibilityQuery
{
    FGraphEventRef TaskEvent;     // タスク完了イベント
    FNaniteVisibilityResults Results;
};

// 可視性管理（フレーム単位でクエリを管理）
class FNaniteVisibility
{
public:
    // クエリ発行（非同期タスク開始）
    FNaniteVisibilityQuery* BeginVisibilityQuery(
        const TArrayView<const FPrimitiveSceneInfo*>& ScenePrimitives,
        const FNaniteRasterPipelines* RasterPipelines,
        const FNaniteShadingPipelines* ShadingPipelines);

    // 結果取得（タスク完了待ち）
    const FNaniteVisibilityResults* GetVisibilityResults(
        const FNaniteVisibilityQuery* Query) const;

    // タスクイベント取得（依存グラフ構築用）
    FGraphEventRef GetVisibilityTask(
        const FNaniteVisibilityQuery* Query) const;

    // フレーム開始・終了
    void BeginFrame();
    void EndFrame();
};

// スコープドフレーム管理（コンストラクタ=BeginFrame, デストラクタ=EndFrame）
class FNaniteScopedVisibilityFrame
{
    FNaniteScopedVisibilityFrame(FNaniteVisibility& InVisibility);
    ~FNaniteScopedVisibilityFrame();
};

} // namespace Nanite
```

---

## 主要関数

| 関数 | 説明 |
|------|------|
| `BeginVisibilityQuery(...)` | 非同期タスクを開始してクエリを返す |
| `GetVisibilityResults(Query)` | タスク完了まで待機して結果を返す |
| `GetVisibilityTask(Query)` | タスクのグラフイベントを返す（依存設定用） |
| `IsRasterBinVisible(BinIndex)` | 指定ビンが可視かどうか |
| `IsShadingBinVisible(BinIndex)` | 指定シェーディングビンが可視かどうか |
