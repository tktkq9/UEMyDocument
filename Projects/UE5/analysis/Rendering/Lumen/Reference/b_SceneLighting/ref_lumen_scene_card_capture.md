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
    FIntPoint Size;

    FRDGTextureRef Albedo;
    FRDGTextureRef Normal;
    FRDGTextureRef Emissive;
    FRDGTextureRef DepthStencil;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Size` | `FIntPoint` | アトラス解像度（キャプチャ枚数・解像度に応じて可変）|
| `Albedo` | `FRDGTextureRef` | ベースカラー (`PF_R8G8B8A8`) |
| `Normal` | `FRDGTextureRef` | 法線 (`PF_R8G8B8A8`) |
| `Emissive` | `FRDGTextureRef` | エミッシブ (`PF_FloatR11G11B10`) |
| `DepthStencil` | `FRDGTextureRef` | 深度ステンシル (`PF_DepthStencil`) |

### 使用箇所
- [[ref_lumen_scene_card_capture]] `LumenScene::AllocateCardCaptureAtlas()` — 確保
- [[ref_lumen_scene_card_capture]] `AddCardCaptureDraws()` — RT としてバインド
- [[ref_lumen_scene_rendering]] `UpdateLumenSurfaceCacheAtlas()` — Surface Cache アトラスへのコピー元

---

## FResampledCardCaptureAtlas

前フレームの照明を **リサンプリング** して再利用するための一時アトラス。  
Card の移動・変形後にバウンス光を再利用してちらつきを抑える。

```cpp
struct FResampledCardCaptureAtlas {
    FIntPoint Size;

    FRDGTextureRef DirectLighting;
    FRDGTextureRef IndirectLighting;
    FRDGTextureRef NumFramesAccumulated;
    FRDGBufferRef  TileShadowDownsampleFactor;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Size` | `FIntPoint` | リサンプルアトラスのサイズ（FCardCaptureAtlas.Size と同じ）|
| `DirectLighting` | `FRDGTextureRef` | 前フレームからリサンプルした Direct Lighting |
| `IndirectLighting` | `FRDGTextureRef` | 前フレームからリサンプルした Indirect Lighting |
| `NumFramesAccumulated` | `FRDGTextureRef` | 各テクセルが何フレーム蓄積されているか（テンポラルウェイト用）|
| `TileShadowDownsampleFactor` | `FRDGBufferRef` | タイルごとのシャドウダウンサンプル係数（シャドウ計算コスト削減用）|

### 使用箇所
- [[ref_lumen_scene_lighting]] `FCopyCardCaptureLightingToAtlasPS` — `bResampleLastLighting = true` 時に参照

---

## FCardPageRenderData

1 枚の Card ページをキャプチャするために必要な全描画データ。  
`AddCardCaptureDraws()` で生成され、Nanite / 非 Nanite のメッシュバッチを保持する。

```cpp
class FCardPageRenderData {
public:
    int32 PrimitiveGroupIndex;
    int32 CardIndex;
    int32 PageTableIndex;

    FVector4f CardUVRect;
    FIntRect  CardCaptureAtlasRect;
    FIntRect  SurfaceCacheAtlasRect;

    FLumenCardOBBd CardWorldOBB;
    FViewMatrices  ViewMatrices;

    TArray<uint32>            NaniteInstanceIds;
    TArray<FNaniteShadingBin> NaniteShadingBins;

    bool bHeightField;
    bool bResampleLastLighting;
    bool bAxisXFlipped;

    void UpdateViewMatrices(const FViewInfo& MainView);
    void PatchView(const FScene* Scene, FViewInfo* CardView) const;
    bool HasNanite() const;
    bool NeedsRender() const;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `PrimitiveGroupIndex` | `int32` | 所属 `FLumenPrimitiveGroup` のシーンインデックス |
| `CardIndex` | `int32` | `FLumenSceneData::Cards` 内の Card インデックス |
| `PageTableIndex` | `int32` | `FLumenSceneData::PageTable` 内のエントリインデックス |
| `CardUVRect` | `FVector4f` | Card 全体内での UV 範囲（ResLevel 解像度に対する比率）|
| `CardCaptureAtlasRect` | `FIntRect` | 一時キャプチャアトラス上の描画領域（px）|
| `SurfaceCacheAtlasRect` | `FIntRect` | Surface Cache 物理アトラス上の書き込み先（px）|
| `CardWorldOBB` | `FLumenCardOBBd` | Card の World 空間 OBB（キャプチャカメラ計算用）|
| `ViewMatrices` | `FViewMatrices` | キャプチャ用ビュー行列（正投影、`UpdateViewMatrices()` で生成）|
| `NaniteInstanceIds` | `TArray<uint32>` | キャプチャ対象の Nanite インスタンス ID 配列 |
| `NaniteShadingBins` | `TArray<FNaniteShadingBin>` | Nanite シェーディングビン（マテリアル分類）|
| `bHeightField` | `bool` | Heightfield 由来の Card か（描画パスを切り替え）|
| `bResampleLastLighting` | `bool` | 前フレーム照明をリサンプルするか（Card が移動した場合は false）|
| `bAxisXFlipped` | `bool` | X 軸が反転しているか（裏面 Card の場合）|

---

## FCardPageRenderData::UpdateViewMatrices

```cpp
void FCardPageRenderData::UpdateViewMatrices(const FViewInfo& MainView);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `MainView` | `const FViewInfo&` | メインビュー（参照のみ、実際には正投影を構築）|

### 使用箇所
- [[ref_lumen_scene_card_capture]] `AddCardCaptureDraws()` — FCardPageRenderData 構築後に呼ばれる

### 内部処理フロー

1. **カメラ位置の決定（Card OBB の正面から正投影）**
   ```cpp
   FVector AxisX = CardWorldOBB.AxisX;
   FVector AxisY = CardWorldOBB.AxisY;
   FVector AxisZ = CardWorldOBB.AxisZ; // Card の法線方向
   FVector FaceLocalExtent = CardWorldOBB.Extent;

   FVector ViewLocation = CardWorldOBB.Origin + FaceLocalExtent.Z * AxisZ;
   ```

2. **Near / Far プレーンの設定**
   ```cpp
   float NearPlane = 0.0f;
   float FarPlane  = FaceLocalExtent.Z * 2.0f;
   float ZScale    = 1.0f / (FarPlane - NearPlane);
   ```

3. **投影矩形の計算（CardUVRect → NDC）**
   ```cpp
   // CardUVRect [0,1] → [-1,1] のNDC空間
   float ProjectionL = CardUVRect.X * 2.0f - 1.0f;
   float ProjectionR = CardUVRect.Z * 2.0f - 1.0f;
   float ProjectionB = CardUVRect.Y * 2.0f - 1.0f;
   float ProjectionT = CardUVRect.W * 2.0f - 1.0f;
   ```

4. **正交投影行列の構築（Reversed-Z）**
   ```cpp
   FMatrix ProjectionMatrix = FReversedZOrthoMatrix(
       ProjectionL, ProjectionR,
       ProjectionB, ProjectionT,
       ZScale, 0.0f);
   ```

5. **ビュー回転行列（AxisXYZ から構築）**
   ```cpp
   FMatrix ViewRotationMatrix = FMatrix(
       FPlane(AxisX.X, AxisY.X, -AxisZ.X, 0),
       FPlane(AxisX.Y, AxisY.Y, -AxisZ.Y, 0),
       FPlane(AxisX.Z, AxisY.Z, -AxisZ.Z, 0),
       FPlane(0, 0, 0, 1));
   ViewMatrices = FViewMatrices(ViewLocation, ViewRotationMatrix, ProjectionMatrix);
   ```

---

## FCardPageRenderData::PatchView

```cpp
void FCardPageRenderData::PatchView(const FScene* Scene, FViewInfo* CardView) const;
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `Scene` | `const FScene*` | シーン（`SetupUniformBufferParameters()` に渡す）|
| `CardView` | `FViewInfo*` | 更新対象のビュー情報（インプレース更新）|

### 使用箇所
- [[ref_lumen_scene_card_capture]] キャプチャパスの描画前に Card 用 FViewInfo を生成する

### 内部処理フロー

```cpp
void FCardPageRenderData::PatchView(const FScene* Scene, FViewInfo* CardView) const
{
    CardView->ProjectionMatrixUnadjustedForRHI = ViewMatrices.GetProjectionMatrix();
    CardView->ViewMatrices = ViewMatrices;
    CardView->ViewRect = CardCaptureAtlasRect;
    CardView->NearClippingDistance = 0.0f;
    CardView->SetupUniformBufferParameters(Scene, ...);
    // ViewUniformBuffer を再生成して Card 用のビュー空間に切り替え
}
```

---

> [!note]- HasNanite / NeedsRender — 判定ヘルパー
> 
> ```cpp
> bool FCardPageRenderData::HasNanite() const {
>     return NaniteInstanceIds.Num() > 0;
> }
> bool FCardPageRenderData::NeedsRender() const {
>     return HasNanite() || VisibleMeshDrawCommands.Num() > 0;
> }
> ```
> 
> **使用箇所**: [[ref_lumen_scene_card_capture]] `AddCardCaptureDraws()` — 描画コマンドが空の Card をスキップ

---

## LumenScene 名前空間（CardCapture 関連）

### AllocateCardCaptureAtlas

```cpp
namespace LumenScene {
    FCardCaptureAtlas AllocateCardCaptureAtlas(
        FRDGBuilder& GraphBuilder,
        const TArray<FCardPageRenderData>& CardPagesToRender);
}
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `CardPagesToRender` | `const TArray<FCardPageRenderData>&` | キャプチャ対象 Card ページのリスト |

#### 使用箇所
- [[ref_lumen_scene_rendering]] `UpdateLumenScene()` — キャプチャパス冒頭

#### 内部処理フロー

```cpp
FCardCaptureAtlas LumenScene::AllocateCardCaptureAtlas(...)
{
    // 全 CardPageRenderData の CardCaptureAtlasRect を包む最小矩形
    FIntPoint TotalSize = ComputeAtlasSize(CardPagesToRender);

    FCardCaptureAtlas Atlas;
    Atlas.Size = TotalSize;
    Atlas.Albedo       = GraphBuilder.CreateTexture(FRDGTextureDesc::Create2D(
        TotalSize, PF_R8G8B8A8, ...), TEXT("Lumen.CardCaptureAtlasAlbedo"));
    Atlas.Normal       = GraphBuilder.CreateTexture(...PF_R8G8B8A8..., "Normal");
    Atlas.Emissive     = GraphBuilder.CreateTexture(...PF_FloatR11G11B10..., "Emissive");
    Atlas.DepthStencil = GraphBuilder.CreateTexture(...PF_DepthStencil..., "Depth");
    return Atlas;
}
```

---

### AddCardCaptureDraws

```cpp
namespace LumenScene {
    void AddCardCaptureDraws(
        const FScene* Scene,
        const FSceneViewFamily& ViewFamily,
        const FLumenSceneFrameTemporaries& FrameTemporaries,
        TArray<FCardPageRenderData>& CardPagesToRender,
        FMeshCommandOneFrameArray& VisibleMeshDrawCommands,
        TArray<int32>& PrimitiveIds);
}
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `Scene` | `const FScene*` | シーン（PrimitiveSceneInfo 参照）|
| `ViewFamily` | `const FSceneViewFamily&` | ビューファミリ |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | PrimitiveGroupBuffer など |
| `CardPagesToRender` | `TArray<FCardPageRenderData>&` | 入出力: Card ページリスト（Nanite ID なども書き込む）|
| `VisibleMeshDrawCommands` | `FMeshCommandOneFrameArray&` | 出力: 非 Nanite メッシュの描画コマンド |
| `PrimitiveIds` | `TArray<int32>&` | 出力: 処理したプリミティブ ID |

#### 使用箇所
- [[ref_lumen_scene_rendering]] `UpdateLumenScene()` — キャプチャアトラス確保後

#### 内部処理フロー

1. **各 Card ページについてプリミティブの検索**
   ```cpp
   for (FCardPageRenderData& PageData : CardPagesToRender) {
       const FLumenPrimitiveGroup& PrimGroup =
           LumenData.PrimitiveGroups[PageData.PrimitiveGroupIndex];
       for (FPrimitiveSceneInfo* PrimSceneInfo : PrimGroup.Primitives) {
   ```

2. **Nanite メッシュの処理**
   ```cpp
           if (IsNaniteMesh(PrimSceneInfo)) {
               PageData.NaniteInstanceIds.Add(PrimSceneInfo->GetNaniteInstanceId());
               if (bLandscape) {
                   // ランドスケープ用: SourceComponentIds から ShadingBin を検索
               }
               PageData.NaniteShadingBins.Add(GetNaniteShadingBin(PrimSceneInfo));
   ```

3. **非 Nanite メッシュの LOD 選択**
   ```cpp
           } else {
               // TargetScreenSize に基づいて LOD を選択
               float TargetScreenSize = CVarLumenSceneSurfaceCacheMeshTargetScreenSize; // 0.15
               int32 LODIndex = SelectLOD(PrimSceneInfo->LODTable, TargetScreenSize);
               // ランドスケープは固定 LOD: LumenCardCapture::LandscapeLOD (= 0)

               for (FStaticMesh& StaticMesh : PrimSceneInfo->StaticMeshes) {
                   if (StaticMesh.LODIndex == LODIndex && StaticMesh.bUseForMaterial) {
                       VisibleMeshDrawCommands.Add(
                           BuildMeshDrawCommand(StaticMesh, PageData.CardCaptureAtlasRect, ...));
                   }
               }
           }
       }
   }
   ```

---

> [!note]- HasPrimitiveNaniteMeshBatches — Nanite バッチ有無の確認
> 
> ```cpp
> bool LumenScene::HasPrimitiveNaniteMeshBatches(const FPrimitiveSceneProxy* Proxy);
> ```
> 
> **内部動作**: `Proxy->IsNaniteMesh() && Proxy->DoesProjectCastDynamicShadow()`
> 
> **使用箇所**: `AddCardCaptureDraws()` — プリミティブが Nanite パスで処理されるか判定

---

> [!note]- AllowSurfaceCacheCardSharing — Card 共有の許可
> 
> ```cpp
> bool LumenScene::AllowSurfaceCacheCardSharing();
> ```
> 
> **内部動作**: `r.LumenScene.SurfaceCache.CardSharing != 0`
> 
> **目的**: 同一スタティックメッシュの複数インスタンス間で Card を共有し、キャプチャコストを削減

---

> [!note]- CullUndergroundTexels — 地下テクセルカリング
> 
> ```cpp
> bool LumenScene::CullUndergroundTexels();
> ```
> 
> **内部動作**: `r.LumenScene.SurfaceCache.CullUndergroundTexels != 0`
> 
> **目的**: World の Y 軸=0 より下のテクセルをシャドウレイトレースから除外して負荷削減

---

## シェーダークラス（マイナー）

> [!note]- FLumenCardVS — メッシュマテリアルキャプチャ VS
> 
> ```cpp
> class FLumenCardVS : public FMeshMaterialShader {
>     static bool ShouldCompilePermutation(const FMeshMaterialShaderPermutationParameters& Params);
> };
> ```
> 
> **説明**: Lumen カードキャプチャ用のメッシュマテリアル VS。`SupportsLumenMeshCards()` な頂点ファクトリが対象。
> 
> **使用箇所**: [[ref_lumen_scene_card_capture]] 非 Nanite メッシュのキャプチャ描画

---

> [!note]- FLumenCardPS — メッシュマテリアルキャプチャ PS
> 
> ```cpp
> class FLumenCardPS : public FMeshMaterialShader {
>     static void ModifyCompilationEnvironment(
>         const FMaterialShaderPermutationParameters& Params,
>         FShaderCompilerEnvironment& OutEnvironment);
> };
> ```
> 
> **コンパイルフラグ**:
> - `SUBSTRATE_INLINE_SHADING = 1`
> - `SUBSTRATE_USE_FULLYSIMPLIFIED_MATERIAL = 1`
> - `SCENE_TEXTURES_DISABLED = 1`
> 
> **使用箇所**: [[ref_lumen_scene_card_capture]] 非 Nanite メッシュのキャプチャ描画（マテリアル評価）

---

> [!note]- FLumenCardCS — Nanite コンピュートキャプチャシェーダー
> 
> ```cpp
> class FLumenCardCS : public FMeshMaterialShader {
>     static void ModifyCompilationEnvironment(...);
>     static FShaderPermutationRequirements GetPermutationRequirements();
>     virtual EShaderPriority GetOverrideJobPriority() const override; // ExtraHigh
> };
> ```
> 
> **コンパイルフラグ**: `CFLAG_ForceDXC | CFLAG_HLSL2021 | CFLAG_RootConstants | CFLAG_SupportsMinimalBindless`
> 
> **使用箇所**: Nanite 経由のカードキャプチャ（`Nanite::DrawLumenCapturePasses()` から呼ばれる）

---

## キャプチャパイプライン全体フロー

```
UpdateLumenScene() → BuildCardUpdateList() で更新 Card ページを選定
  │
  ├─ LumenScene::AllocateCardCaptureAtlas()
  │     → FCardCaptureAtlas (Albedo/Normal/Emissive/Depth) を RDG で確保
  │
  ├─ LumenScene::AddCardCaptureDraws()
  │     ├─ [非 Nanite] FMeshDrawCommandPassSetup + SetupCardCaptureRenderTargetsInfo
  │     │     → FLumenCardVS + FLumenCardPS で描画
  │     └─ [Nanite]    Nanite::DrawLumenCapturePasses() + FLumenCardCS
  │
  ├─ FClearLumenCardsPS  → 新規割り当て Card ページをアトラスにクリア（黒塗り）
  │
  ├─ FCopyCardCaptureLightingToAtlasPS
  │     ├─ bResampleLastLighting = true  → 前フレーム照明をリサンプルして書き込み
  │     └─ bResampleLastLighting = false → 0 初期化
  │
  └─ UpdateLumenSurfaceCacheAtlas()
        → Albedo / Normal / Emissive を Surface Cache アトラスにコピー
```

---

## 主要 CVar（LumenSceneCardCapture.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.SurfaceCache.MeshTargetScreenSize` | 0.15 | LOD 選択の基準スクリーンサイズ |
| `r.LumenScene.SurfaceCache.NaniteLODScaleFactor` | 1.0 | Nanite の LOD スケール係数 |
| `r.LumenScene.SurfaceCache.NaniteLandscapeLODScaleFactor` | 1.0 | Nanite ランドスケープの LOD スケール |
| `r.LumenScene.SurfaceCache.Nanite.MultiView` | 1 | Nanite MultiView キャプチャを使うか |
| `r.LumenScene.SurfaceCache.ResampleLighting` | 1 | 照明リサンプリングを有効にするか |
| `r.LumenScene.SurfaceCache.CardCapturesPerFrame` | 300 | 1 フレームあたりの最大キャプチャ Card 数 |
| `r.LumenScene.SurfaceCache.CardCaptureFactor` | 64 | キャプチャテクセル上限の係数 |
| `r.LumenScene.SurfaceCache.CardCaptureRefreshFraction` | 0.125 | 既存 Card ページ再キャプチャ割合 |
