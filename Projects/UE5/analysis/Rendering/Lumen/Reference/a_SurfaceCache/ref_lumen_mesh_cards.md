# リファレンス：LumenMeshCards.h / LumenMeshCards.cpp

- グループ: a - Surface Cache
- 上位: [[a_lumen_surface_cache]]
- 関連: [[ref_lumen_scene_data]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenMeshCards.h/cpp`

---

## 概要

1 メッシュに紐づく **Card（平面パッチ）群** を管理するクラス。  
メッシュの AABB を 6 方向に投影して Card を生成・更新する処理を担う。

---

## クラス・関数一覧

### FLumenMeshCards

```cpp
class FLumenMeshCards {
public:
    // 初期化
    void Initialize(
        const FMatrix& InLocalToWorld,
        int32 InPrimitiveGroupIndex,
        uint32 InFirstCardIndex,
        uint32 InNumCards,
        const FMeshCardsBuildData& MeshCardsBuildData,
        const FLumenPrimitiveGroup& PrimitiveGroup);

    // ルックアップテーブルを更新（Card が追加/削除された後に呼ぶ）
    void UpdateLookup(const TSparseSpanArray<FLumenCard>& Cards);

    // トランスフォームを更新
    void SetTransform(const FMatrix& InLocalToWorld);

    // ワールド空間 AABB を計算して返す
    FBox GetWorldSpaceBounds() const;

    // ---- メンバ変数 ----
    FMatrix LocalToWorld;
    FVector3f LocalToWorldScale;
    FMatrix WorldToLocalRotation;
    FBox LocalBounds;

    int32 PrimitiveGroupIndex = -1;
    bool bFarField = false;           // 遠距離フィールド扱いか
    bool bHeightfield = false;        // ハイトフィールドか
    bool bMostlyTwoSided = false;     // 両面マテリアルが多いか
    bool bEmissiveLightSource = false; // 自発光光源か

    uint32 FirstCardIndex = 0;        // Card 配列の先頭インデックス
    uint32 NumCards = 0;              // Card 数
    // 6 方向（±X/Y/Z）→ Card インデックスのルックアップ
    uint32 CardLookup[Lumen::NumAxisAlignedDirections]; // NumAxisAlignedDirections = 6

    TArray<int32, TInlineAllocator<1>> ScenePrimitiveIndices;
};
```

### LumenMeshCards 名前空間

```cpp
namespace LumenMeshCards {
    // Card の最小表面積しきい値を返す
    // bEmissiveLightSource == true の場合は小さいしきい値（自発光は小さい Card も有効）
    float GetCardMinSurfaceArea(bool bEmissiveLightSource);
}
```

### Lumen 名前空間（MeshCards.h）

```cpp
namespace Lumen {
    constexpr uint32 NumAxisAlignedDirections = 6; // ±X/Y/Z の 6 方向

    // Card Scene バッファの更新（GPU バッファに MeshCards データをアップロード）
    void UpdateCardSceneBuffer(
        FRDGBuilder& GraphBuilder,
        FRDGScatterUploadBuilder& UploadBuilder,
        FLumenSceneFrameTemporaries& FrameTemporaries,
        const FSceneViewFamily& ViewFamily,
        FScene* Scene);
}
```

---

## CardLookup の仕組み

```
方向インデックス（AxisAlignedDirectionIndex）:
  0 = +X
  1 = -X
  2 = +Y
  3 = -Y
  4 = +Z
  5 = -Z

CardLookup[方向インデックス] → その方向の FLumenCard のインデックス
                              (有効な Card がない場合は INDEX_NONE)
```

---

## 主要 CVar（LumenScene.cpp で定義）

```
r.LumenScene.GlobalSDF.Resolution = 252
    → Global Distance Field の解像度（Lumen 有効時）

r.LumenScene.GlobalSDF.ClipmapExtent = 2500.0
    → 最初のクリップマップの半径（ワールド単位）

r.LumenScene.SurfaceCache.AtlasSize = 4096
    → Surface Cache アトラスサイズ（px）

r.LumenScene.FarField = 0
    → 遠距離フィールド有効/無効（変更時にプロキシ再作成が走る）

r.LumenScene.FarField.OcclusionOnly = 0
    → 遠距離フィールドをオクルージョンのみにして高速化

r.LumenScene.FarField.MaxTraceDistance = 1.0e6
    → 遠距離フィールドの最大トレース距離

r.LumenScene.FarField.FarFieldDitherScale = 200.0
    → 近距離/遠距離の境界ディザリング幅（ワールド単位）
```
