# VSM a: VirtualShadowMapArray（コア管理）

- 対象: `VirtualShadowMaps/VirtualShadowMapArray.h/.cpp`
- 上位: [[04_vsm_overview]]

---

## 役割

`FVirtualShadowMapArray` は **1フレームごとに生成される VSM 全体の管理クラス**。  
`FDeferredShadingRenderer` が保持し、ページテーブル・物理ページプール・描画パスを統括する。

---

## ページング構造

```
仮想アドレス空間: 16K × 16K テクセル
  = Level0DimPagesXY(128) × PageSize(128テクセル)

物理ページプール: Texture2DArray (可変枚数)
  各スロットが 1 物理ページ(128×128テクセル)に対応

PageTable: 仮想ページ → 物理ページインデックスのマッピング
PageFlags: 各ページの状態（ALLOCATED / CACHED / DIRTY 等）
```

---

## ライト種別ごとのVSM割り当て

| ライト種別 | 割り当て関数 | 備考 |
|-----------|------------|------|
| ディレクショナル | `AllocateDirectional(Count)` | クリップマップの各レベルに1つずつ |
| ローカル（フル） | `AllocateLocal(false, Count)` | ポイント/スポット等 |
| ローカル（シングルページ） | `AllocateLocal(true, Count)` | 小さいライト用の軽量モード |
| 未参照（オフスクリーン） | `AllocateUnreferenced(Count)` | キャッシュ維持のみ |

---

## フレーム内の主要処理順序

```
Initialize()
  → CacheManager から物理ページプール・前フレームデータを受け取る

AllocateDirectional() / AllocateLocal()
  → ライトごとに VirtualShadowMapId を割り当て

BeginMarkPages()
  → GBuffer / シャドウレシーバー情報をもとに
    どの仮想ページが必要かフラグを立てる
  → Froxel（体積照明）・半透明フロントレイヤーも対象

BuildPageAllocations()
  → 要求フラグが立ったページに物理ページを割り当て
  → 静的キャッシュ済みページはスキップ（差分更新）

RenderVirtualShadowMapsNanite()
  → Nanite ジオメトリを VSM 用ビューとして深度ラスタライズ
  → FNaniteVirtualShadowMapRenderPass でビュー配列を渡す

RenderVirtualShadowMapsNonNanite()
  → 非Naniteメッシュを深度専用パスで描画

UpdateHZB()
  → 物理ページプールのHZBを再構築
  → 次フレームのオクルージョンカリングに使用

PostRender()
  → フレームデータをCacheManagerへ引き渡し
```

---

## Nanite 統合

```cpp
// Nanite 用の VSM レンダーパス記述子
struct FNaniteVirtualShadowMapRenderPass
{
    // Nanite の汎用ビュー配列として VSM 用ビューを渡す
    Nanite::FPackedViewArray* VirtualShadowMapViews;
    // 対象シャドウリスト
    TArray<FProjectedShadowInfo*> Shadows;
    // Nanite カリングクエリ
    FSceneInstanceCullingQuery* SceneInstanceCullingQuery;
    // ビュー数上限
    int32 TotalPrimaryViews;
    int32 MaxCullingViews;
};
```

Nanite は `EPipeline::Shadows` モードでラスタライズし、  
結果を VSM の物理ページプールに直接書き込む。

---

## 主要バッファ一覧

| バッファ名 | 型 | 役割 |
|-----------|---|------|
| `PhysicalPagePoolRDG` | Texture2DArray | 実シャドウデプスデータ |
| `HZBPhysicalArrayRDG` | Texture2DArray | 物理ページ用HZB |
| `PageTableRDG` | Texture2D uint | 仮想→物理マッピング |
| `PageRequestFlagsRDG` | Buffer | ページ要求フラグ（BeginMarkPages用）|
| `PageFlagsRDG` | Buffer | ページ確定フラグ |
| `PageReceiverMasksRDG` | Buffer | レシーバーライティングチャンネルマスク |
| `ProjectionDataRDG` | StructuredBuffer | ライトごとの投影行列等 |
| `AllocatedPageRectBoundsRDG` | Buffer | 割り当てページのAABB |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Shadow.Virtual.Enable` | 1 | VSM 全体有効/無効 |
| `r.Shadow.Virtual.MaxPhysicalPages` | 2048 | 物理ページプール上限 |
| `r.Shadow.Virtual.UseHZB` | 1 | 二パスHZBオクルージョン |
| `r.Shadow.Virtual.MarkPixelPages` | 1 | ピクセルベースページマーキング |
| `r.Shadow.Virtual.MarkCoarsePagesLocal` | 2 | ローカルライト粗ページ |
| `r.Shadow.Virtual.MarkCoarsePagesDirectional` | 1 | ディレクショナル粗ページ |
| `r.Shadow.Virtual.PageDilationBorderSizeLocal` | 0.05 | ローカルページ拡張ボーダー |
| `r.Shadow.Virtual.PageDilationBorderSizeDirectional` | 0.05 | ディレクショナルページ拡張ボーダー |
| `r.Shadow.Virtual.NormalBias` | 0.5 | 法線バイアス |
| `r.Shadow.Virtual.NonNanite.IncludeInCoarsePages` | 1 | 非Naniteを粗ページに含めるか |
| `r.Shadow.Virtual.NonNanite.UseHZB` | 1 | 非Nanite HZBオクルージョン |
| `r.Shadow.Virtual.DynamicRes.MaxPagePoolLoadFactor` | 0.85 | 動的解像度ページプール負荷係数 |
