# Nanite Debug & Editor（デバッグ・エディタ）

- 上位: [[03_nanite_overview]]
- 関連: [[a_nanite_cull_raster]] | [[b_nanite_materials_shading]]

---

## 概要

Nanite の開発支援・デバッグ機能群：

- **NaniteVisualize**: r.Nanite.Visualize による各種デバッグビューとピッキング
- **NaniteEditor**: エディタ上でのヒットプロキシ描画・選択ハイライト
- **NaniteFeedback**: バッファオーバーフロー検出・マテリアルパフォーマンス警告

---

## NaniteVisualize — デバッグビューモード

```cpp
// デバッグビュー種別（r.Nanite.Visualize の値に対応）
enum class EDebugViewMode : uint8
{
    None,
    Wireframe,         // ワイヤーフレーム表示
    ClusterID,         // クラスターID
    PageID,            // ページID
    GroupID,           // グループID
    MaterialID,        // マテリアルID
    ShadingBin,        // シェーディングビン
    RasterBin,         // ラスタライズビン
    TriangleCount,     // 三角形密度
    LevelInstance,     // レベルインスタンス境界
    ShaderComplexity,  // シェーダー複雑度
    NaniteOverdraw,    // Nanite オーバードロー
    // ...
};

// ビジュアライゼーションパスの追加
void AddVisualizationPasses(
    FRDGBuilder& GraphBuilder,
    const FScene& Scene,
    const FViewInfo& View,
    const FRasterResults& RasterResults,
    ...);

// ピッキング結果の表示（マウスカーソル下のクラスター情報）
void DisplayPicking(
    FRDGBuilder& GraphBuilder,
    const FNanitePickingFeedback& PickingFeedback,
    ...);
```

---

## NaniteEditor — エディタ選択

```cpp
// ヒットプロキシ描画（エディタクリック判定用）
void DrawHitProxies(
    FRDGBuilder& GraphBuilder,
    const FScene& Scene,
    const FViewInfo& View,
    FRasterResults& RasterResults,
    FRDGTextureRef HitProxyTexture,
    FRDGTextureRef HitProxyDepthTexture);

// 選択ハイライト描画
void DrawEditorSelection(
    FRDGBuilder& GraphBuilder,
    const FScene& Scene,
    const FViewInfo& View,
    ...);

// レベルインスタンスの可視化
void DrawEditorVisualizeLevelInstance(
    FRDGBuilder& GraphBuilder,
    const FScene& Scene,
    const FViewInfo& View,
    ...);
```

---

## NaniteFeedback — パフォーマンスフィードバック

```cpp
// フィードバック管理（バッファ状態・警告）
class FFeedbackManager
{
public:
    // マテリアルパフォーマンス警告の報告
    void ReportMaterialPerformanceWarning(
        const FMaterialWarningItem& Item);

    // バッファオーバーフロー発生時のメッセージID
    FName GetStatusMessageId() const;

private:
    // バッファ状態追跡
    FBufferState BufferState;

    // 警告アイテム一覧
    TArray<FMaterialWarningItem> MaterialWarnings;
};

// バッファオーバーフロー検出: クラスター数・ビン数が上限を超えたら警告を表示
// GPUStat: NaniteEditor（エディタ専用ステータス）
```

---

## 関連ソースファイル

| ファイル | 役割 |
|---------|------|
| `NaniteVisualize.h/.cpp` | r.Nanite.Visualize によるデバッグビュー・ピッキング表示 |
| `NaniteEditor.h/.cpp` | ヒットプロキシ描画・エディタ選択・レベルインスタンス可視化 |
| `NaniteFeedback.h/.cpp` | バッファオーバーフロー検出・マテリアル性能警告 |

---

## 関連リファレンス

| リファレンス | 対象ソース | 主な内容 |
|------------|---------|---------|
| [[ref_nanite_visualize]] | `NaniteVisualize.h/.cpp` | EDebugViewMode / AddVisualizationPasses / DisplayPicking |
| [[ref_nanite_editor]] | `NaniteEditor.h/.cpp` | DrawHitProxies / DrawEditorSelection / DrawEditorVisualizeLevelInstance |
| [[ref_nanite_feedback]] | `NaniteFeedback.h/.cpp` | FFeedbackManager / FBufferState / ReportMaterialPerformanceWarning |
