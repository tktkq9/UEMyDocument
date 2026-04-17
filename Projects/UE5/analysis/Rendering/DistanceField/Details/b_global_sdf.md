# B: Global SDF — クリップマップ構造・UpdateGlobalDistanceFieldVolume

- 対象: `GlobalDistanceField.h/.cpp`, `GlobalDistanceFieldParameters.h`, `SceneRendering.h:627`
- 上位: [[10_distance_field_overview]]
- Reference: [[ref_global_sdf]]

---

## 概要

**Global SDF** はシーン全体のワールド空間 Signed Distance Field をクリップマップ（多段 LOD ボリューム）として管理する。  
Mesh SDF の内容をランタイムで合成し、カメラに追従しながら差分更新される。  
DFAO のコーントレースと Lumen の SurfaceCache トレースが主な利用先。

---

## FGlobalDFCacheType（キャッシュ種別）

`GlobalDistanceField.h:18`

```cpp
enum FGlobalDFCacheType
{
    GDF_MostlyStatic,  // 静的オブジェクトのみを含むクリップマップ
    GDF_Full,          // 静的 + 動的オブジェクトを含む完全クリップマップ
    GDF_Num            // 種別数（2）
};
```

| キャッシュ種別 | 内容 | 更新頻度 |
|-------------|------|---------|
| `GDF_MostlyStatic` | 移動しないスタティックメッシュのみ | 変化があった部分のみ差分更新 |
| `GDF_Full` | 動的オブジェクトも含む（毎フレーム合成） | 毎フレーム（カメラ追従部分）|

---

## クリップマップ構成

```
Global SDF クリップマップ（Lumen 無効時：4 段、有効時：最大 8 段）

  Clipmap[0]  最近傍（高解像度・狭範囲）
  Clipmap[1]  中距離
  Clipmap[2]  遠距離
  Clipmap[3]  最遠（低解像度・広範囲）
  ...

  カメラ移動に追従してスライディング更新
  カメラカット・大規模変更時は全面再構築
```

**クリップマップ解像度・範囲（GlobalDistanceField namespace 関数）：**

| 関数 | 説明 |
|-----|------|
| `GetClipmapResolution(bLumenEnabled)` | 各クリップマップのボクセル解像度 |
| `GetClipmapExtent(index, Scene, bLumenEnabled)` | クリップマップの空間的なカバー範囲（半径） |
| `GetNumGlobalDistanceFieldClipmaps(bLumenEnabled, LumenSceneViewDistance)` | クリップマップ段数（4 または 8+）|
| `GetMipFactor()` | Mip 間のスケール比 |

---

## FGlobalDistanceFieldInfo（ランタイム状態）

`SceneRendering.h:627`

```cpp
class FGlobalDistanceFieldInfo
{
public:
    bool bInitialized = false;

    TArray<FGlobalDistanceFieldClipmap> MostlyStaticClipmaps; // GDF_MostlyStatic 段
    TArray<FGlobalDistanceFieldClipmap> Clipmaps;             // GDF_Full 段

    FGlobalDistanceFieldParameterData ParameterData; // シェーダーバインドデータ

    // GPU テクスチャ（ページベースのスパース構造）
    TRefCountPtr<FRDGPooledBuffer>    PageFreeListAllocatorBuffer;
    TRefCountPtr<FRDGPooledBuffer>    PageFreeListBuffer;
    TRefCountPtr<IPooledRenderTarget> PageAtlasTexture;         // SDF 値を格納
    TRefCountPtr<IPooledRenderTarget> CoverageAtlasTexture;     // カバレッジ情報
    TRefCountPtr<FRDGPooledBuffer>    PageObjectGridBuffer;
    TRefCountPtr<IPooledRenderTarget> PageTableCombinedTexture; // ページテーブル（統合）
    TRefCountPtr<IPooledRenderTarget> PageTableLayerTextures[GDF_Num]; // 種別別ページテーブル
    TRefCountPtr<IPooledRenderTarget> MipTexture;               // MIP テクスチャ
};
```

---

## FGlobalDistanceFieldParameterData（シェーダーバインド）

`GlobalDistanceFieldParameters.h:17`

```cpp
class FGlobalDistanceFieldParameterData
{
    FVector4f TranslatedCenterAndExtent[MaxClipmaps];       // クリップマップ中心・範囲
    FVector4f TranslatedWorldToUVAddAndMul[MaxClipmaps];    // ワールド→UV 変換
    FVector4f MipTranslatedWorldToUVScale[MaxClipmaps];     // Mip テクスチャ UV スケール
    FVector4f MipTranslatedWorldToUVBias[MaxClipmaps];      // Mip テクスチャ UV バイアス
    float     MipFactor;
    float     MipTransition;

    FRHITexture* PageAtlasTexture;    // Texture3D: SDF ページアトラス
    FRHITexture* CoverageAtlasTexture;// Texture3D: カバレッジアトラス
    FRHITexture* PageTableTexture;    // Texture3D<uint>: ページテーブル
    FRHITexture* MipTexture;          // Texture3D: Mip SDF

    int32        ClipmapSizeInPages;  // クリップマップ1段のページ数
    FVector      InvPageAtlasSize;    // 1 / PageAtlasSize
    FVector      InvCoverageAtlasSize;
    int32        MaxPageNum;
    float        GlobalDFResolution;  // ボクセル解像度
    float        MaxDFAOConeDistance; // DFAO コーントレース最大距離
    int32        NumGlobalSDFClipmaps;
};
```

---

## UpdateGlobalDistanceFieldVolume

`GlobalDistanceField.h:50` / `GlobalDistanceField.cpp`

```cpp
extern void UpdateGlobalDistanceFieldVolume(
    FRDGBuilder& GraphBuilder,
    FRDGExternalAccessQueue& ExternalAccessQueue,
    FViewInfo& View,
    FScene* Scene,
    float MaxOcclusionDistance,   // DFAO の最大遮蔽距離
    bool bLumenEnabled,           // Lumen 使用時はクリップマップ段数が増える
    FGlobalDistanceFieldInfo& Info);
```

**処理フロー：**

```
UpdateGlobalDistanceFieldVolume()
  │
  ├─ カメラ移動量を計算
  │   ├─ 小移動 → 差分更新（スライディング）
  │   └─ 大移動 / カメラカット → 全面更新フラグ
  │
  ├─ PendingAddOperations から Mesh SDF を各クリップマップへ合成
  │   └─ ComposeObjects CS（GlobalDistanceField.usf）
  │
  ├─ GDF_MostlyStatic クリップマップ更新
  │   → 静的オブジェクトのみ合成・キャッシュ
  │
  └─ GDF_Full クリップマップ更新
      → GDF_MostlyStatic をベースに動的オブジェクトを合成
```

---

## DFAO vs Lumen での利用の違い

| 利用先 | 参照する SDF | 目的 |
|-------|------------|------|
| **DFAO** | `GDF_Full` クリップマップ | コーントレースで AO 計算（Sky Light 遮蔽） |
| **Lumen SurfaceCache** | `GDF_Full` クリップマップ + `GDF_MostlyStatic` | HW/SW レイトレースの補助・Secondary Bounce |
| **Lumen Screen Probe** | Global SDF | 遠距離の粗いレイマーチ |

Lumen が有効な場合、DFAO は自動的に無効化される（`ShouldRenderDistanceFieldAO()` 内で判定）。

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.AOGlobalDistanceField` | Global SDF の有効/無効（0: 無効, 1: 有効） |
| `r.GlobalDistanceField.MaxPageNum` | ページアトラスの最大ページ数 |
| `r.GlobalDistanceField.ClipmapExtent` | クリップマップの範囲係数 |
| `r.Lumen.SceneViewDistance` | Lumen 有効時のクリップマップ段数に影響 |

---

## コード実行フロー（呼び出し元）

```
FDeferredShadingSceneRenderer::Render()  [DeferredShadingRenderer.cpp]
  │
  └─ UpdateGlobalDistanceFieldVolume(GraphBuilder, ExternalAccessQueue, View, Scene, ...)
       │
       ├─ ClipmaapUpdateBounds を計算（差分更新領域）
       ├─ ComposeObjectsIntoClipmap CS × クリップマップ数
       │   → GlobalDistanceField.usf
       └─ FGlobalDistanceFieldInfo::ParameterData を更新
            → 以後、DFAO / Lumen がシェーダーパラメータとして利用
```
