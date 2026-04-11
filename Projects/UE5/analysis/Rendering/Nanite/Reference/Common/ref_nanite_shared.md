# リファレンス：NaniteShared.h / NaniteShared.cpp

- グループ: Common
- 上位: [[03_nanite_overview]]
- 関連: [[ref_nanite_core]] | [[ref_nanite_cull_raster]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteShared.h/.cpp`

---

## 概要

Nanite 全体で共有される型定義・ビュー管理・シェーダー基底クラス・グローバルリソースを定義するファイル。  
`FPackedView` によるビューデータのパック化と、  
`FGlobalResources` による GPU リソースの永続管理が中心。

---

## 主要クラス・構造体

```cpp
// パック化されたビューデータ（GPU バッファに直接アップロードする形式）
struct FPackedView
{
    FMatrix44f  SVPositionToTranslatedWorld; // SVPosition → ワールド変換行列
    FMatrix44f  ViewToTranslatedWorld;
    FVector4f   ViewRect;                    // (MinX, MinY, MaxX, MaxY)
    FVector4f   ViewSizeAndInvSize;
    FVector4f   ClipSpaceScaleOffset;
    FVector4f   PreViewTranslation;
    // LOD・カリング用パラメータ
    float       LODScale;
    float       MinBoundsRadiusSq;           // 最小バウンド半径（カリング用）
    uint32      StreamingPriorityCategory;   // ストリーミング優先度
    // ... 他多数のパック済みフィールド
};

// 複数ビューの配列（マルチビューパス用）
struct FPackedViewArray
{
    TArray<FPackedView> Views;
    uint32 NumPrimaryViews;    // プライマリビュー数
    uint32 NumMipViews;        // MIP ビュー数（シャドウカスケード等）

    // ビュー配列の RDG バッファを取得
    FRDGBufferRef GetOrCreateBuffer(FRDGBuilder& GraphBuilder);
};

// ビューパラメータ（FPackedView の元データ）
struct FPackedViewParams
{
    const FSceneView* SceneView;
    FIntRect ViewRect;
    float LODScaleFactor;
    uint32 StreamingPriorityCategory;
    // ...
};

// GPU グローバルリソース（永続バッファ）
class FGlobalResources : public FRenderResource
{
public:
    // 各種定数バッファ・構造体バッファ
    FRWBufferStructured StatsBuffer;      // GPU 統計バッファ
    FRWBufferStructured MainPassBuffers;  // Main Pass 用バッファ
    FRWBufferStructured PostPassBuffers;  // Post Pass 用バッファ

    // ページテーブル関連
    FRWBufferStructured StreamingRequests; // ストリーミングリクエスト
};

// Nanite 専用グローバルシェーダー基底クラス
class FNaniteGlobalShader : public FGlobalShader
{
    // ShouldCompilePermutation: Nanite サポートプラットフォームのみコンパイル
    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters&);
};

// Nanite 専用マテリアルシェーダー基底クラス
class FNaniteMaterialShader : public FMaterialShader
{
    static bool ShouldCompilePermutation(const FMaterialShaderPermutationParameters&);
};

// ラスタライズパイプライン定義
struct FNaniteRasterPipeline
{
    FNaniteRasterBin PrimaryBin;     // 通常ビン
    FNaniteRasterBin SecondaryBin;   // セカンダリビン（両面等）
    bool bIsMaterialMasked;          // マスク有効か
    bool bForceDisableWPO;           // WPO 強制無効か
};

// ラスタライズビン（マテリアル別の Raster パイプライン単位）
struct FNaniteRasterBin
{
    int32 BinIndex;       // ビンID（-1 = 無効）
    uint16 MaterialFlags; // Masked / TwoSided / PDO 等
};
```

---

## FPackedView — 主要フィールド

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `SVPositionToTranslatedWorld` | `FMatrix44f` | SVPosition → ワールド変換行列 |
| `ViewToTranslatedWorld` | `FMatrix44f` | ビュー → ワールド変換行列 |
| `ViewRect` | `FVector4f` | (MinX, MinY, MaxX, MaxY) |
| `ViewSizeAndInvSize` | `FVector4f` | ビューサイズと逆数 |
| `ClipSpaceScaleOffset` | `FVector4f` | クリップ空間スケール・オフセット |
| `PreViewTranslation` | `FVector4f` | 浮動小数点精度補正用平行移動 |
| `LODScale` | `float` | LOD スケール係数 |
| `MinBoundsRadiusSq` | `float` | 最小バウンド半径²（カリング用）|
| `StreamingPriorityCategory` | `uint32` | ストリーミング優先度カテゴリ |

---

## 主要関数

| 関数 | 引数 | 説明 |
|------|------|------|
| `CreatePackedView()` | `FPackedViewParams` | ビューデータをパック化して返す |
| `CreatePackedViewArray()` | `TArray<FPackedViewParams>` | 複数ビューの配列を作成 |
| `SetCullingViewOverrides()` | `FPackedView&`, override params | カリングビューパラメータを上書き |
| `ShouldDrawSceneViewsInOneNanitePass()` | `FSceneViewFamily` | 複数ビューを1パスで描画できるか判定 |

---

> [!note]- FPackedView の GPU アップロード
> `FPackedViewArray::GetOrCreateBuffer()` が RDG バッファを作成し、  
> GPU 上でカリングシェーダーが直接インデックスアクセスできる形式でアップロードする。  
> VR / マルチビューの場合、`FPackedViewArray::Views` に複数ビューが入り、  
> `ShouldDrawSceneViewsInOneNanitePass()` が true であれば1回の DrawGeometry() で処理される。

> [!note]- FNaniteGlobalShader vs FNaniteMaterialShader
> `FNaniteGlobalShader` はカリング・ビニング・VisBuffer読み取り等の固定処理シェーダーに使う。  
> `FNaniteMaterialShader` はマテリアル固有のシェーディング（GBuffer 書き込み）に使う。  
> 両クラスの `ShouldCompilePermutation()` が Nanite 対応プラットフォーム（SM5以上）のみを選別する。

> [!note]- FGlobalResources の永続性
> `FGlobalResources` は `FRenderResource` を継承するシングルトン的リソース。  
> `InitRasterContext()` で毎フレーム作るフレーム一時バッファとは異なり、  
> `StatsBuffer` / `StreamingRequests` 等はシーンのライフタイムにわたって維持される。  
> `InitRHI()` でバッファを確保し `ReleaseRHI()` で解放する。
