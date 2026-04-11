# リファレンス：NaniteCullRaster.h / NaniteCullRaster.cpp

- グループ: a - CullRaster
- 上位: [[a_nanite_cull_raster]]
- 関連: [[ref_nanite_composition]] | [[ref_nanite_shared]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteCullRaster.h/.cpp`

---

## 概要

Nanite の中核。GPU Driven な2段階クラスターカリングと、  
HW / SW（Compute）ハイブリッドラスタライズを統括する。  
`InitRasterContext()` でコンテキストを準備し、  
`CullRasterize()` 相当の処理でインスタンス→クラスター→三角形の順に絞り込む。

---

## 主要クラス・列挙型

```cpp
namespace Nanite
{

// ラスタライズスケジューリング戦略
enum class ERasterScheduling : uint8
{
    HardwareOnly,                // HW ラスタのみ
    HardwareThenSoftware,        // HW 完了後 SW（同期）
    HardwareAndSoftwareOverlap,  // 非同期コンピュートで HW ∥ SW
};

// 出力バッファモード
enum class EOutputBufferMode : uint8
{
    VisBuffer,    // VisBuffer64（通常パス）
    DepthOnly,    // 深度のみ（シャドウパス）
};

// パイプライン種別（どのレンダリングパスか）
enum class EPipeline : uint8
{
    Primary,   // BasePass
    Shadows,   // シャドウデプス
    Lumen,     // Lumen Surface Cache キャプチャ
    Editor,    // エディタ選択
    HitProxy,  // ヒットプロキシ
};

// ラスタライズコンテキスト（フレームごとに InitRasterContext で作成）
struct FRasterContext
{
    FRDGTextureRef VisBuffer64;         // Depth(40bit) + MaterialDepth(24bit)
    FRDGTextureRef DbgBuffer64;         // デバッグバッファ
    FRDGBufferRef  ShadingMaskBuffer;   // シェーディングマスク（ビン可視判定）
    ERasterScheduling RasterScheduling;
    EOutputBufferMode OutputBufferMode;
    uint32 RasterBinCount;              // 登録済みラスタライズビン数
    bool bCustomPass;                   // カスタムパスか
};

// ラスタライズ結果（カリング完了後に取得）
struct FRasterResults
{
    // ページストリーミング要求（GPU→ストリーミングシステムへ）
    FRDGBufferRef PageRequests;
    // テッセレーション用可視パッチリスト
    FRDGBufferRef VisiblePatches;
    uint32 VisiblePatchesArgs;
    // 可視性テスト結果
    FNaniteVisibilityResults VisibilityResults;
    uint32 ViewsCount;
};

} // namespace Nanite
```

---

## 主要関数

| 関数 | 引数 | 説明 |
|------|------|------|
| `InitRasterContext()` | `FRDGBuilder&`, `FSharedContext&`, `FPackedViewArray&`, ... | RasterContext の初期化・バッファ作成 |
| `CollectRasterPSOInitializers()` | `FSceneTexturesConfig&`, `FMeshPassProcessorRenderState&`, ... | HW ラスタ用 PSO を事前キャッシュ |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Nanite.AsyncRasterization` | 1 | 非同期コンピュートラスタ有効 |
| `r.Nanite.AsyncRasterization.ShadowDepths` | 1 | シャドウパスの非同期化 |
| `r.Nanite.AsyncRasterization.CustomPass` | 1 | カスタムパスの非同期化 |
| `r.Nanite.AsyncRasterization.LumenMeshCards` | 1 | Lumen カードパスの非同期化 |
| `r.Nanite.ComputeRasterization` | 1 | SW（Compute）ラスタ有効 |
| `r.Nanite.ProgrammableRaster` | 1 | マスク・PDO 等プログラマブルラスタ |
| `r.Nanite.MeshShaderRasterization` | 1 | メッシュシェーダー使用（DX12） |
| `r.Nanite.PrimShaderRasterization` | 1 | プリミティブシェーダー使用（AMD） |
| `r.Nanite.Tessellation` | 0 | マイクロポリゴンテッセレーション（実験的） |
| `r.Nanite.MaxPixelsPerEdge` | 1.0 | 目標エッジピクセル数（LOD） |
| `r.Nanite.MinPixelsPerEdgeHW` | 32.0 | HW ラスタ切り替え閾値 |
| `r.Nanite.DicingRate` | 1.0 | マイクロポリゴン目標サイズ |
| `r.Nanite.MaxPatchesPerGroup` | 1 | テッセレーションパッチバッチ上限 |
| `r.Nanite.FilterPrimitives` | 0 | デバッグ用プリミティブフィルタ |
| `r.Nanite.DepthBucketing` | 1 | 深度バケッティング最適化 |

---

> [!note]- 2パスカリングの遅延について
> Pass 1 は前フレームの HZB を使うため、1フレームのカリング遅延が生じる。  
> Post Pass でこの遅延による漏れ（前フレームでは不可視だったが今フレームで可視になったクラスター）を補完する。  
> このため毎フレーム「Pass 1 + Post Pass」の2回カリングが実行されるが、  
> 実際には Pass 1 で大部分が棄却されるため Post Pass のコストは小さい。

> [!note]- EOutputBufferMode::DepthOnly の用途
> シャドウパス（`EPipeline::Shadows`）では GBuffer は不要なため、  
> `EOutputBufferMode::DepthOnly` を指定して深度テクスチャのみを出力する。  
> この場合 `VisBuffer64` は確保されず、MaterialDepth も書き込まれない。

> [!note]- ShadingMaskBuffer の役割
> `ShadingMaskBuffer` は各ラスタービンがラスタライズされたかどうかのビットマスクを記録する。  
> `ShadeBinning()` がこれを参照して、実際にピクセルが書き込まれたビンのみを分類対象にすることで、  
> 空ビンへのシェーディングコストを削減する。
