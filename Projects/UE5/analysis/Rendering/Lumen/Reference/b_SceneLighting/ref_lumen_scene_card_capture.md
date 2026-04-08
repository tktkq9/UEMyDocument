# リファレンス：LumenSceneCardCapture.h / LumenSceneCardCapture.cpp

- グループ: b - Scene Lighting
- 上位: [[b_lumen_scene_lighting]]
- 関連: [[ref_lumen_scene_lighting]] | [[ref_lumen_scene_data]] | [[ref_lumen_mesh_cards]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneCardCapture.h/cpp`

---

## 概要

Surface Cache の **Card キャプチャ**（メッシュ表面の Albedo / Normal / Emissive / Depth を撮影し、  
アトラスに書き込む処理）を担うファイル。  
Nanite / 非 Nanite 両方のメッシュに対応し、新規追加や照明リサンプリングも管理する。

---

## FCardCaptureAtlas

Card キャプチャ用の一時アトラス。1 フレームのキャプチャ結果をここに書き込む。

```cpp
struct FCardCaptureAtlas {
    FIntPoint Size;              // アトラス解像度（可変）

    FRDGTextureRef Albedo;       // ベースカラー (R8G8B8A8)
    FRDGTextureRef Normal;       // 法線 (R8G8B8A8)
    FRDGTextureRef Emissive;     // エミッシブ (R11G11B10F)
    FRDGTextureRef DepthStencil; // 深度ステンシル
};
```

---

## FResampledCardCaptureAtlas

前フレームの照明を **リサンプリング** して再利用するための一時アトラス。  
Card の移動・変形後にバウンス光を再利用してちらつきを抑える。

```cpp
struct FResampledCardCaptureAtlas {
    FIntPoint Size;

    FRDGTextureRef DirectLighting;          // Direct Lighting（リサンプル後）
    FRDGTextureRef IndirectLighting;        // Indirect Lighting（リサンプル後）
    FRDGTextureRef NumFramesAccumulated;    // 各ピクセルの蓄積フレーム数
    FRDGBufferRef  TileShadowDownsampleFactor; // タイルごとのシャドウダウンサンプル係数
};
```

---

## FCardPageRenderData

1 枚の Card ページをキャプチャするために必要な全描画データ。  
`AddCardCaptureDraws()` で生成され、Nanite / 非 Nanite のメッシュバッチを保持する。

```cpp
class FCardPageRenderData {
public:
    // --- インデックス情報 ---
    int32 PrimitiveGroupIndex; // 所属プリミティブグループ
    int32 CardIndex;           // Card インデックス（FLumenSceneData::Cards）
    int32 PageTableIndex;      // ページテーブル内のインデックス

    // --- UV / アトラス矩形 ---
    FVector4f CardUVRect;                // Card 内の UV 範囲
    FIntRect  CardCaptureAtlasRect;      // キャプチャアトラス上の描画領域
    FIntRect  SurfaceCacheAtlasRect;     // Surface Cache アトラス上の配置領域

    // --- カメラ情報 ---
    FLumenCardOBBd CardWorldOBB;         // Card の OBB（World 空間）
    FViewMatrices  ViewMatrices;         // キャプチャ用ビュー行列（正投影）

    // --- Nanite 関連 ---
    TArray<uint32>           NaniteInstanceIds;   // キャプチャ対象の Nanite インスタンスID
    TArray<FNaniteShadingBin> NaniteShadingBins;  // Nanite シェーディングビン

    // --- フラグ ---
    bool bHeightField;           // Heightfield 由来の Card か
    bool bResampleLastLighting;  // 前フレーム照明をリサンプルするか
    bool bAxisXFlipped;          // X 軸が反転しているか（裏面 Card）

    // --- メソッド ---
    void UpdateViewMatrices(const FViewInfo& MainView); // ビュー行列を更新
    void PatchView(const FScene* Scene, FViewInfo* CardView) const; // Card 用 FViewInfo にパッチ
    bool HasNanite() const;   // Nanite インスタンスを持つか
    bool NeedsRender() const; // 描画が必要か（既にキャッシュ済みなら不要）
};
```

---

## LumenScene 名前空間（CardCapture 関連）

| 関数 | 説明 |
|-----|------|
| `HasPrimitiveNaniteMeshBatches(const FPrimitiveSceneProxy*)` | プリミティブに Nanite メッシュバッチがあるか |
| `AllowSurfaceCacheCardSharing()` | Card の共有を許可するか（同一メッシュの複数インスタンス最適化）|
| `CullUndergroundTexels()` | 地下テクセルをカリングするか |
| `AllocateCardCaptureAtlas(GraphBuilder, CardPagesToRender, Atlas)` | キャプチャアトラスを確保 |
| `AddCardCaptureDraws(Scene, ViewFamily, FrameTemporaries, CardPagesToRender, ...)` | 描画コマンドを積む（Nanite / 非 Nanite 分岐）|

---

## キャプチャパイプライン全体フロー

```
UpdateLumenScene() → キャプチャが必要な Card ページを選定
  │
  ├─ AllocateCardCaptureAtlas()
  │     → FCardCaptureAtlas を確保（Albedo/Normal/Emissive/Depth）
  │
  ├─ AddCardCaptureDraws()
  │     ├─ [非 Nanite] FMeshDrawCommandPassSetup でメッシュを描画
  │     └─ [Nanite]    Nanite::DrawLumenCapturePasses() で描画
  │
  ├─ FClearLumenCardsPS → 新規 Card ページをクリア
  │
  ├─ FCopyCardCaptureLightingToAtlasPS
  │     ├─ bResampleLastLighting = true  → 前フレーム照明をリサンプルして書き込み
  │     └─ bResampleLastLighting = false → ゼロ初期化
  │
  └─ UpdateLumenSurfaceCacheAtlas()
        → Albedo / Normal / Emissive を Surface Cache アトラスにコピー
```

---

## 主要 CVar（LumenSceneCardCapture.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.SurfaceCache.CardCapturesPerFrame` | 300 | 1 フレームあたりの最大キャプチャ Card 数 |
| `r.LumenScene.SurfaceCache.CardCaptureFactor` | 64 | キャプチャテクセル上限の係数 |
| `r.LumenScene.SurfaceCache.CardCaptureRefreshFraction` | 0.125 | 既存 Card ページ再キャプチャ割合 |
| `r.LumenScene.SurfaceCache.Nanite.MultiView` | 1 | Nanite MultiView キャプチャを使うか |
| `r.LumenScene.SurfaceCache.ResampleLighting` | 1 | 照明リサンプリングを有効にするか |
