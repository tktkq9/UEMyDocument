# リファレンス：NaniteFeedback.h / NaniteFeedback.cpp

- グループ: f - Debug & Editor
- 上位: [[f_nanite_debug_editor]]
- 関連: [[ref_nanite_visualize]] | [[ref_nanite_cull_raster]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteFeedback.h/.cpp`

---

## 概要

Nanite の GPU バッファオーバーフロー検出とマテリアルパフォーマンス警告を管理するシステム。  
- クラスター数・ビン数等が上限を超えた場合にスクリーンメッセージで警告
- プログラマブルラスタやマスクドマテリアルが多すぎる場合に性能劣化警告を発行

---

## 主要クラス

```cpp
namespace Nanite
{

// フィードバック管理（シングルトン的にシーンに1つ）
class FFeedbackManager
{
public:
    // マテリアルパフォーマンス警告を報告
    void ReportMaterialPerformanceWarning(
        const FMaterialWarningItem& Item);

    // スクリーンメッセージの識別子を取得（更新チェック用）
    FName GetStatusMessageId() const;

    // フレーム開始時にバッファ状態を更新
    void Update(const FBufferState& NewState);
};

// バッファ状態追跡（GPU から読み戻したオーバーフローフラグ等）
struct FBufferState
{
    bool bMainPassOverflow;       // Main Pass クラスターバッファ超過
    bool bPostPassOverflow;       // Post Pass クラスターバッファ超過
    bool bShadingBinOverflow;     // シェーディングビンバッファ超過
    uint32 ClusterCount;          // 処理クラスター数（デバッグ表示用）
};

// マテリアル警告アイテム
struct FMaterialWarningItem
{
    FString MaterialName;
    bool bIsProgrammable;   // プログラマブルラスタ使用中か
    bool bIsMasked;         // マスクドマテリアルか
    bool bHasWPO;           // World Position Offset 使用中か
};

} // namespace Nanite
```

---

## 主要関数

| 関数 | 説明 |
|------|------|
| `Update(FBufferState&)` | GPU から読み戻したフラグでバッファ状態を更新 |
| `ReportMaterialPerformanceWarning(FMaterialWarningItem&)` | マテリアル性能警告を登録 |
| `GetStatusMessageId()` | スクリーンメッセージ識別子を返す |
