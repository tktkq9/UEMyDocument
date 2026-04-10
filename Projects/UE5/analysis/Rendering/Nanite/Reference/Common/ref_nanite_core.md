# リファレンス：Nanite.h / Nanite.cpp

- グループ: Common
- 上位: [[03_nanite_overview]]
- 関連: [[ref_nanite_shared]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/Nanite.h/.cpp`

---

## 概要

Nanite レンダリングパイプライン全体のエントリポイント。  
`FSharedContext`・`FRasterContext`・`FRasterResults` を定義し、  
シャドウマップ・シャドウキューブマップ等の発行関数を宣言する。

---

## 主要構造体

```cpp
namespace Nanite
{

// フレーム共通コンテキスト（全パスで共有する設定・バッファ）
struct FSharedContext
{
    uint32 RenderFlags;               // レンダリングフラグ一式
    ERasterScheduling RasterScheduling; // HW Only / HW+SW / Overlap
    FGlobalShaderMap* ShaderMap;      // シェーダーマップ
    EPipeline Pipeline;               // Primary / Shadows / Lumen 等
};

// ラスタライズコンテキスト（パスごとに作成）
struct FRasterContext
{
    FRDGTextureRef VisBuffer64;        // Depth(40bit) + MaterialDepth(24bit)
    FRDGTextureRef DbgBuffer64;        // デバッグバッファ
    FRDGBufferRef  ShadingMaskBuffer;  // シェーディングマスク
    FRasterScheduling RasterScheduling;
    EOutputBufferMode OutputBufferMode;
    uint32 RasterBinCount;
};

// ラスタライズ結果（カリング後の統計・可視情報）
struct FRasterResults
{
    FRDGBufferRef PageRequests;         // ページストリーミング要求
    FRDGBufferRef VisiblePatches;       // テッセレーション用可視パッチ
    FNaniteVisibilityResults VisibilityResults;
    uint32 ViewsCount;
};

// 設定フラグ
struct FConfiguration
{
    uint32 bUpdateStreaming       : 1; // ストリーミング更新を行うか
    uint32 bSupportsMultiplePasses: 1; // 複数パス対応か
    uint32 bAllowProgrammableRaster: 1; // プログラマブルラスタ許可か
};

// レンダリング実装インターフェース
class IRenderer
{
public:
    virtual void ExtractResults(FRasterResults& OutResults) = 0;
    virtual void DrawGeometry(...) = 0;
};

} // namespace Nanite
```

---

## 主要関数

| 関数 | 説明 |
|------|------|
| `Nanite::EmitShadowMap(...)` | Nanite プリミティブのシャドウデプスを生成 |
| `Nanite::EmitCubemapShadow(...)` | キューブマップシャドウを生成 |
| `Nanite::PrintStats(...)` | GPU 統計情報をスクリーンに出力 |
| `Nanite::ExtractShadingDebug(...)` | シェーディングデバッグ情報を抽出 |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Nanite.ShowStats` | 0 | GPU 統計情報の画面表示 |
| `r.Nanite.StatsFilter` | -1 | 統計フィルタ（-1=全て） |
| `r.Nanite.Shadows` | 1 | Nanite シャドウ有効/無効 |
