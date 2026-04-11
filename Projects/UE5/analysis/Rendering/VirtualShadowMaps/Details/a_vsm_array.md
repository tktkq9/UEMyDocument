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

## コード実行フロー

### エントリポイント

```
FDeferredShadingSceneRenderer::Render()
  └─ FVirtualShadowMapArray::Initialize()          VirtualShadowMapArray.cpp:815
       ├─ CacheManager::SetPhysicalPoolSize()      VirtualShadowMapCacheManager.cpp:1137
       ├─ AllocateDirectional() / AllocateLocal()
       │
       ├─ BeginMarkPages()                         VirtualShadowMapArray.cpp:2106
       │    ├─ FMarkPagesByDepthCS (ピクセル別ページマーキング)
       │    └─ FMarkCoarsePagesByFrustumCS (粗ページマーキング)
       │
       ├─ BuildPageAllocations()                   VirtualShadowMapArray.cpp:2818
       │    ├─ AllocateNewPageMappingsCS (新規ページ割り当て)
       │    └─ PropagateMappedMipsCS (MIPマッピング伝播)
       │
       ├─ RenderVirtualShadowMapsNanite()          VirtualShadowMapArray.cpp:3789
       │    └─ Nanite::IRenderer::Create → DrawGeometry (EPipeline::Shadows)
       │
       ├─ RenderVirtualShadowMapsNonNanite()       VirtualShadowMapArray.cpp:3956
       │    └─ 深度専用パスでメッシュ描画
       │
       ├─ UpdateHZB()                              VirtualShadowMapArray.cpp:4395
       │
       └─ PostRender()                             VirtualShadowMapArray.cpp:1729
            └─ ExtractFrameData() → CacheManager に引き渡し
```

### フロー詳細

1. **Initialize()** — `CacheManager` から物理ページプールと前フレームデータを受け取り、フレーム用 RDG リソースを初期化する
2. **BeginMarkPages()** — GBuffer の深度・法線を参照し、各ピクセルがどの仮想ページを必要とするかフラグを立てる。Froxel（体積照明）や半透明も対象
3. **BuildPageAllocations()** — フラグが立った仮想ページに物理ページを割り当て。静的キャッシュ済みページはスキップし差分のみ更新する
4. **RenderVirtualShadowMapsNanite()** — `EPipeline::Shadows` / `EOutputBufferMode::DepthOnly` モードで Nanite カリング・ラスタライズを実行。物理ページプールに直接書き込む
5. **RenderVirtualShadowMapsNonNanite()** — Nanite 以外のメッシュを深度専用パスで描画
6. **UpdateHZB()** — 次フレームのオクルージョンカリング用に物理ページプールの HZB を再構築する
7. **PostRender()** — フレームデータ（PageTable / PageFlags / ProjectionData）を `CacheManager::ExtractFrameData()` で永続バッファに保存する

### 関与クラス・関数一覧

| クラス/関数 | ファイル:行 | 役割 |
|------------|-----------|------|
| `FVirtualShadowMapArray::Initialize()` | `VirtualShadowMapArray.cpp:815` | フレーム初期化 |
| `FVirtualShadowMapArray::BeginMarkPages()` | `VirtualShadowMapArray.cpp:2106` | ページ要求フラグ生成 |
| `FVirtualShadowMapArray::BuildPageAllocations()` | `VirtualShadowMapArray.cpp:2818` | 物理ページ割り当て |
| `FVirtualShadowMapArray::RenderVirtualShadowMapsNanite()` | `VirtualShadowMapArray.cpp:3789` | Nanite 深度描画 |
| `FVirtualShadowMapArray::RenderVirtualShadowMapsNonNanite()` | `VirtualShadowMapArray.cpp:3956` | 非Nanite 深度描画 |
| `FVirtualShadowMapArray::UpdateHZB()` | `VirtualShadowMapArray.cpp:4395` | 物理ページ HZB 更新 |
| `FVirtualShadowMapArray::PostRender()` | `VirtualShadowMapArray.cpp:1729` | フレーム後処理・データ永続化 |

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

---

## 関連リファレンス

- [[ref_vsm_array]] — FVirtualShadowMapArray / FVirtualShadowMapProjectionShaderData リファレンス
- [[ref_vsm_cache_manager]] — FVirtualShadowMapArrayCacheManager リファレンス
- [[ref_vsm_shaders]] — FVirtualShadowMapGlobalShader 基底クラス
