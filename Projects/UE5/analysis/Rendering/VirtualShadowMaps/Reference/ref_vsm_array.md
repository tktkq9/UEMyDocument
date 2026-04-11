# REF: VirtualShadowMapArray.h / .cpp

- 対象ファイル: `Private/VirtualShadowMaps/VirtualShadowMapArray.h` / `.cpp`
- 関連Details: [[a_vsm_array]]

---

## クラス・構造体

### `FVirtualShadowMap`（静的ユーティリティ）

定数のみを保持するユーティリティ名前空間的クラス。

| 定数 | 値 | 説明 |
|-----|---|------|
| `PageSize` | 128 | ページのテクセルサイズ |
| `PageSizeMask` | 127 | ビットマスク（PageSize-1） |
| `Log2PageSize` | 7 | log2(128) |
| `Level0DimPagesXY` | 128 | レベル0のページ数 → 仮想解像度16K×16K |
| `MaxMipLevels` | 8 | MIPレベル数上限 |
| `VirtualMaxResolutionXY` | 16384 | 仮想最大解像度（テクセル） |
| `PhysicalPageAddressBits` | 12 | 物理ページアドレスビット数 |
| `NumHZBLevels` | 7 | HZBレベル数 |

---

### `FNaniteVirtualShadowMapRenderPass`

Nanite で VSM 用に描画する際のパス記述子。

| フィールド | 型 | 説明 |
|-----------|---|------|
| `VirtualShadowMapViews` | `Nanite::FPackedViewArray*` | VSM 用のNaniteビュー配列 |
| `Shadows` | `TArray<FProjectedShadowInfo*>` | このパスで描画するシャドウリスト |
| `SceneInstanceCullingQuery` | `FSceneInstanceCullingQuery*` | カリングクエリ |
| `TotalPrimaryViews` | `int32` | プライマリビュー総数 |
| `MaxCullingViews` | `int32` | カリングビュー上限 |

---

### `FVirtualShadowMapProjectionShaderData`

シェーダーに渡す投影データ（ライト1つ分）。

| フィールド | 型 | 説明 |
|-----------|---|------|
| `ShadowViewToClipMatrix` | `FMatrix44f` | シャドウビュー→クリップ行列 |
| `TranslatedWorldToShadowUVMatrix` | `FMatrix44f` | ワールド→シャドウUV |
| `TranslatedWorldToShadowUVNormalMatrix` | `FMatrix44f` | 法線変換用 |
| `LightDirection` | `FVector3f` | 光源方向 |
| `LightType` | `uint32` | ライトタイプ |
| `PreViewTranslationHigh/Low` | `FVector3f` | プリビュー変換（高精度分割） |
| `LightRadius` | `float` | 光源半径 |
| `ResolutionLodBias` | `float` | 解像度LODバイアス |
| `ClipmapCornerRelativeOffset` | `FVector2f` | クリップマップ原点オフセット |
| `ClipmapLevel_ClipmapLevelCountRemaining` | `int32` | 正=クリップマップレベル, -1=ローカルライト |
| `LightId` | `int32` | ライトID |
| `TexelDitherScale` | `float` | SMRTディザースケール |

---

### `FVirtualShadowMapArray`

フレームごとに生成される VSM 全体の管理クラス。`FDeferredShadingRenderer` が保持。

**主要メソッド:**

| メソッド | 説明 |
|---------|------|
| `Initialize(FRDGBuilder&, FScene&, FViewFamilyInfo&, ...)` | 初期化、CacheManagerからリソース取得 |
| `AllocateDirectional(int32 Count)` | ディレクショナルライト用VSM ID割り当て |
| `AllocateLocal(bool bSinglePage, int32 Count)` | ローカルライト用VSM ID割り当て |
| `AllocateUnreferenced(int32 Count)` | オフスクリーンライト用（キャッシュ維持） |
| `UpdateNextData(int32 VsmId, ...)` | 次フレーム用データ更新 |
| `BeginMarkPages(FRDGBuilder&, ...)` | ページ要求フラグ生成 |
| `BuildPageAllocations(FRDGBuilder&, ...)` | 物理ページ割り当て |
| `RenderVirtualShadowMapsNanite(...)` | Naniteジオメトリ描画 |
| `RenderVirtualShadowMapsNonNanite(...)` | 非Naniteジオメトリ描画 |
| `UpdateHZB(FRDGBuilder&)` | 物理ページプールのHZB再構築 |
| `PostRender(FRDGBuilder&, ...)` | フレーム後処理・CacheManagerへデータ引き渡し |
| `GetSamplingParameters(FRDGBuilder&, int32 ViewIndex)` | シェーダー用サンプリングパラメータ |
| `GetMarkingParameters(FRDGBuilder&)` | ページマーキング用パラメータ |
| `GetUniformBuffer()` | VSMシェーダーパラメータのUniformBuffer |
| `IsAllocated() const` | 物理ページプール割り当て確認 |
| `ShouldCacheStaticSeparately() const` | 静的キャッシュ分離モード確認 |
| `CreateMipViews(TArray<Nanite::FPackedView>&, ...)` | VSMビューのMIPビュー生成 |
| `AddRenderViews(const FVirtualShadowMapClipmap*, ...)` | クリップマップ用レンダービュー追加 |

**主要バッファフィールド:**

| フィールド | 型 | 説明 |
|-----------|---|------|
| `PhysicalPagePoolRDG` | `FRDGTextureRef` | 実シャドウデプス（Texture2DArray） |
| `HZBPhysicalArrayRDG` | `FRDGTextureRef` | 物理ページHZB |
| `PageTableRDG` | `FRDGTextureRef` | 仮想→物理マッピング |
| `PageRequestFlagsRDG` | `FRDGBufferRef` | ページ要求フラグ |
| `PageFlagsRDG` | `FRDGBufferRef` | ページ状態フラグ |
| `PageReceiverMasksRDG` | `FRDGBufferRef` | レシーバーライティングチャンネルマスク |
| `ProjectionDataRDG` | `FRDGBufferRef` | FVirtualShadowMapProjectionShaderData配列 |
| `AllocatedPageRectBoundsRDG` | `FRDGBufferRef` | 割り当てページのAABB |
| `UncachedPageRectBoundsRDG` | `FRDGBufferRef` | 非キャッシュページのAABB |
| `CachedPageInfosRDG` | `FRDGBufferRef` | キャッシュページ情報 |
| `PhysicalPageListsRDG` | `FRDGBufferRef` | 物理ページリスト |

---

### `FVirtualShadowMapVisualizeLightSearch`

可視化ツール用のライト検索ヘルパー。

| メソッド | 説明 |
|---------|------|
| `CheckLight(FLightSceneProxy*, int32 VsmId)` | ライトのVSM IDチェック |
| `ChooseLight(FLightSceneProxy*, int32 VsmId)` | 可視化対象ライトを選択 |
| `IsValid() const` | 有効なライトが見つかったか |

---

## 主要 enum

```cpp
enum class EVSMVisualizationPostPass
{
    PreEditorPrimitives,   // エディタプリミティブ描画前
    PostEditorPrimitives,  // エディタプリミティブ描画後
};

// デバッグ診断スロット数
constexpr int32 MaxPageAreaDiagnosticSlots = 32;
constexpr int32 MaxNPFDiagnosticSlotsSinglePage = 32;
constexpr int32 MaxNPFDiagnosticSlotsMultiPage = 96;
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Shadow.Virtual.Enable` | 1 | VSM 有効/無効 |
| `r.Shadow.Virtual.MaxPhysicalPages` | 2048 | 物理ページプール上限 |
| `r.Shadow.Virtual.UseHZB` | 1 | 二パスHZBオクルージョン |
| `r.Shadow.Virtual.ForceFullHZBUpdate` | 0 | 全HZB強制更新（デバッグ） |
| `r.Shadow.Virtual.MarkPixelPages` | 1 | ピクセルベースページマーキング |
| `r.Shadow.Virtual.MarkPixelPagesMipModeLocal` | 0 | ローカルライトMIPモード |
| `r.Shadow.Virtual.MarkCoarsePagesLocal` | 2 | ローカル粗ページマーキング |
| `r.Shadow.Virtual.MarkCoarsePagesDirectional` | 1 | ディレクショナル粗ページ |
| `r.Shadow.Virtual.PageMarkingPixelStrideX` | 2 | ページマーキングXストライド |
| `r.Shadow.Virtual.PageMarkingPixelStrideY` | 2 | ページマーキングYストライド |
| `r.Shadow.Virtual.PageDilationBorderSizeLocal` | 0.05 | ローカルページ拡張 |
| `r.Shadow.Virtual.PageDilationBorderSizeDirectional` | 0.05 | ディレクショナルページ拡張 |
| `r.Shadow.Virtual.NormalBias` | 0.5 | 法線バイアス |
| `r.Shadow.Virtual.CullBackfacingPixels` | 1 | 裏面ピクセルカリング |
| `r.Shadow.Virtual.ResolutionLodBias` | 0 | 解像度LODバイアス（全体） |
| `r.Shadow.Virtual.NonNaniteVSM` | 1 | 非NaniteメッシュのVSM対応 |
| `r.Shadow.Virtual.NonNanite.IncludeInCoarsePages` | 1 | 非Naniteを粗ページに含める |
| `r.Shadow.Virtual.NonNanite.UseHZB` | 1 | 非NaniteでHZBカリング |
| `r.Shadow.Virtual.NonNanite.Batch` | 1 | 非Naniteのバッチ描画 |
| `r.Shadow.Virtual.DynamicHZB` | 0 | 動的HZBの分離 |
| `r.Shadow.Virtual.DynamicRes.MaxPagePoolLoadFactor` | 0.85 | 動的解像度ページプール負荷係数 |
| `r.Shadow.Virtual.DynamicRes.TimeBudgetMs` | 0 | フレーム時間予算（0=無効） |
| `r.Shadow.Virtual.Stats.Visible` | 0 | デバッグ統計表示 |
| `r.Shadow.Virtual.Visualize.Layout` | 0 | 可視化レイアウト |
| `r.Shadow.Virtual.Visualize.LightName` | "" | 可視化対象ライト名 |

---

> [!note]- FVirtualShadowMapArray のライフサイクル（フレームスコープ）
> `FVirtualShadowMapArray` は `FDeferredShadingRenderer` のメンバとしてフレームごとに生成・破棄される一時オブジェクト。  
> 物理ページプールなどの永続リソースは所有せず、`FVirtualShadowMapArrayCacheManager`（`ISceneExtension`）が保持する。  
> `Initialize()` でリソースを借り受け、`PostRender()` でフレームデータを `CacheManager` に返す。

> [!note]- ClipmapLevel_ClipmapLevelCountRemaining の符号による判別
> `FVirtualShadowMapProjectionShaderData::ClipmapLevel_ClipmapLevelCountRemaining` は正の値の場合クリップマップレベルを表し、  
> -1 の場合はローカルライトを表す。シェーダー内でこの符号を見てディレクショナル/ローカルを判別している。  
> クリップマップレベル番号（8〜18）はワールド空間の半径スケールに直結し、レベルが1増えるごとに半径が2倍になる。

> [!note]- RenderVirtualShadowMapsNanite と EPipeline::Shadows
> `RenderVirtualShadowMapsNanite()` は `Nanite::IRenderer::Create()` に `FSharedContext::Pipeline = EPipeline::Shadows` と  
> `EOutputBufferMode::DepthOnly` を渡して実行する。これにより VisBuffer64 は確保されず、  
> シャドウデプスのみが VSM 物理ページプールの該当スロットに直接書き込まれる。  
> クリップマップの各レベルが個別の `FPackedView` として Nanite ビュー配列に入り、1回の DrawGeometry() で全レベルを処理できる。
