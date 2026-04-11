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

## コード実行フロー

### デバッグビジュアライゼーション フロー

```
FDeferredShadingSceneRenderer::RenderNanite()
  │  FRasterResults 取得後
  │
  └─ [View.CurrentNaniteVisualizationMode が None 以外の場合]
       Nanite::AddVisualizationPasses()
         → r.Nanite.Visualize の値に対応する EDebugViewMode を選択
         → VisBuffer64 / DbgBuffer64 からデバッグ情報を抽出
         → スクリーン上にオーバーレイ描画
         → (Picking が有効な場合) DisplayPicking()
              → マウスカーソル下のクラスター情報を GPU から読み出して表示
```

### エディタ選択 フロー

```
FDeferredShadingSceneRenderer::RenderHitProxies()
  └─ Nanite::DrawHitProxies()
       → EPipeline::HitProxy モードで IRenderer::Create()
       → DrawGeometry() → VisBuffer（ヒットプロキシID版）
       → HitProxyTexture に書き込み
       → エディタがクリック位置の Actor を特定

FDeferredShadingSceneRenderer::Render() [エディタビルド]
  └─ Nanite::DrawEditorSelection()
       → 選択された Nanite プリミティブにアウトラインオーバーレイ
  └─ Nanite::DrawEditorVisualizeLevelInstance()
       → レベルインスタンスの境界ボックスを描画
```

### フィードバック フロー

```
RenderNanite() 実行後
  └─ Nanite::FFeedbackManager::Update()
       → GPU から統計バッファ（クラスター数・ビン数）を非同期で読み返し
       → 上限（r.Nanite.MaxClusters 等）を超えた場合に
         スクリーンに警告メッセージを表示（GetStatusMessageId()）
       → マテリアル警告（プログラマブルラスタ多用等）を
         ReportMaterialPerformanceWarning() でログ出力
```

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 説明 |
|--------------|---------|------|
| `Nanite::AddVisualizationPasses()` | `NaniteVisualize.h` | デバッグビューオーバーレイの追加 |
| `Nanite::DisplayPicking()` | `NaniteVisualize.h` | マウス位置のクラスター情報表示 |
| `Nanite::DrawHitProxies()` | `NaniteEditor.h` | エディタクリック判定用 HitProxy 描画 |
| `Nanite::DrawEditorSelection()` | `NaniteEditor.h` | 選択アウトラインオーバーレイ |
| `Nanite::DrawEditorVisualizeLevelInstance()` | `NaniteEditor.h` | レベルインスタンス境界描画 |
| `Nanite::FFeedbackManager::Update()` | `NaniteFeedback.h` | GPU 統計フィードバック更新・警告表示 |

---

## 関連リファレンス

| リファレンス | 対象ソース | 主な内容 |
|------------|---------|---------|
| [[ref_nanite_visualize]] | `NaniteVisualize.h/.cpp` | EDebugViewMode / AddVisualizationPasses / DisplayPicking |
| [[ref_nanite_editor]] | `NaniteEditor.h/.cpp` | DrawHitProxies / DrawEditorSelection / DrawEditorVisualizeLevelInstance |
| [[ref_nanite_feedback]] | `NaniteFeedback.h/.cpp` | FFeedbackManager / FBufferState / ReportMaterialPerformanceWarning |
