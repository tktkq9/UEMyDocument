# VirtualShadowMaps ソースマップ

- 対象: Virtual Shadow Maps（仮想化シャドウマップ）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[04_vsm_overview]]

16K×16K 仮想アドレス空間をページング（128×128 テクセル単位）。
フレーム単位の `FVirtualShadowMapArray` と永続 `FVirtualShadowMapArrayCacheManager` が協調。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/VirtualShadowMaps/` |
| シェーダー | `Engine/Shaders/Private/VirtualShadowMaps/*.usf` |

---

## ファイル → クラス対応

### フレーム管理

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `VirtualShadowMapArray.h` | `FVirtualShadowMapArray`, `FVirtualShadowMapUniformParameters`, `FVirtualShadowMap` (定数) | フレームごとの VSM 管理 | [[Reference/ref_vsm_array]] |
| `VirtualShadowMapArray.cpp` | `Initialize()`:815, `AllocateDirectional()/AllocateLocal()`, `BeginMarkPages()`:2106, `BuildPageAllocations()`:2818, `RenderVirtualShadowMapsNanite()`:3789, `RenderVirtualShadowMapsNonNanite()`:3956, `UpdateHZB()`:4395, `PostRender()`:1729 | ページ割り当て・描画・HZB 更新 | [[Details/a_vsm_array]] |

### キャッシュ

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `VirtualShadowMapCacheManager.h` | `FVirtualShadowMapArrayCacheManager`, `FVirtualShadowMapCacheEntry` | フレーム間キャッシュ・LRU・静的ページ保持 | [[Reference/ref_vsm_cache_manager]] |
| `VirtualShadowMapCacheManager.cpp` | `ProcessInvalidations()`:1883, `ExtractFrameData()`:1456 | シーン変更による無効化・永続化 | [[Details/b_vsm_cache_manager]] |

### Clipmap（Directional）

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `VirtualShadowMapClipmap.h/.cpp` | `FVirtualShadowMapClipmap`, `GetLevelShaderParameters()` | ディレクショナルライト用複数レベル clipmap | [[Reference/ref_vsm_clipmap]], [[Details/c_vsm_clipmap]] |

### Projection（SMRT）

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `VirtualShadowMapProjection.h/.cpp` | `RenderVirtualShadowMapProjection()`:437, `FVirtualShadowMapProjectionShaderData` | SMRT（Stochastic Micro Ray Tracing）によるシャドウマスク生成 | [[Reference/ref_vsm_projection]], [[Details/d_vsm_projection]] |

### シェーダー定義

| ファイル | 役割 |
|---------|------|
| `VirtualShadowMapShaders.h` | 共通シェーダー定義・定数 |
| `VirtualShadowMapDefinitions.h` | `FVirtualShadowMap::PageSize=128`, `Level0DimPagesXY=128`, `MaxMipLevels=8` |
| `Shaders/Private/VirtualShadowMaps/*.usf` | Page Mark / Build / Projection / HZB |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─[A] FVirtualShadowMapArray::Initialize             :815
  │      └─ CacheManager から物理ページプール・前フレームデータ受領
  │
  ├─[B] AllocateDirectional / AllocateLocal
  │      └─ ライトごとに VirtualShadowMapId 割り当て
  │
  ├─[C] BeginMarkPages                                 :2106
  │      └─ GBuffer・Shadow Receiver から必要ページフラグ
  │
  ├─[D] BuildPageAllocations                           :2818
  │      └─ 要求フラグ → 物理ページ割り当て（静的キャッシュはスキップ）
  │
  ├─[E] RenderVirtualShadowMapsNanite                  :3789
  │      └─ Nanite::IRenderer::Create(EPipeline::Shadows) → DrawGeometry
  │
  ├─[F] RenderVirtualShadowMapsNonNanite               :3956
  │      └─ 通常メッシュを深度専用パスで描画
  │
  ├─[G] UpdateHZB                                      :4395
  │      └─ 物理ページプールの HZB 再構築
  │
  ├─[H] RenderVirtualShadowMapProjection               :437
  │      └─ SMRT でシャドウマスク生成
  │
  └─[I] PostRender                                     :1729
         └─ ExtractFrameData → CacheManager
```

---

## 主要構造体

| 構造体 | 場所 |
|-------|------|
| `FVirtualShadowMapUniformParameters` | `VirtualShadowMapArray.h` |
| `FVirtualShadowMapProjectionShaderData` | `VirtualShadowMapProjection.h` |
| `FNaniteVirtualShadowMapRenderPass` | `VirtualShadowMapArray.h`（Nanite ビュー配列連携）|

VSM の RDG テクスチャ:
- `PhysicalPagePoolRDG`（Texture2DArray uint）— 実データ
- `PageTableRDG`（Texture2D uint）— 仮想→物理マッピング
- `PageFlagsRDG` / `PageRequestFlagsRDG`

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.Shadow.Virtual.Enable` | `VirtualShadowMapArray.cpp` |
| `r.Shadow.Virtual.ResolutionLodBias` | `VirtualShadowMapArray.cpp` |
| `r.Shadow.Virtual.Cache.StaticSeparate` | `VirtualShadowMapCacheManager.cpp` |
| `r.Shadow.Virtual.SMRT.RayCountLocal` | `VirtualShadowMapProjection.cpp` |
| `r.Shadow.Virtual.SMRT.RayCountDirectional` | `VirtualShadowMapProjection.cpp` |
| `r.Shadow.Virtual.NonNanite.IncludeInCoarsePages` | `VirtualShadowMapArray.cpp` |
| `r.Shadow.Virtual.MaxPhysicalPages` | `VirtualShadowMapArray.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_vsm_array]] | FVirtualShadowMapArray |
| Reference | [[Reference/ref_vsm_cache_manager]] | FVirtualShadowMapArrayCacheManager |
| Reference | [[Reference/ref_vsm_clipmap]] | FVirtualShadowMapClipmap |
| Reference | [[Reference/ref_vsm_projection]] | RenderVirtualShadowMapProjection |
| Reference | [[Reference/ref_vsm_shaders]] | シェーダー定義 |
| Details | [[Details/a_vsm_array]] | ページ割り当て・描画詳細 |
| Details | [[Details/b_vsm_cache_manager]] | キャッシュ無効化・LRU |
| Details | [[Details/c_vsm_clipmap]] | Clipmap レベル構成 |
| Details | [[Details/d_vsm_projection]] | SMRT 詳細 |

---

## ue5-dive 起点

- 「どのページが要求されるか」 → `VirtualShadowMapArray.cpp:BeginMarkPages:2106`
- 「静的キャッシュのヒット判定」 → `VirtualShadowMapCacheManager.cpp:ProcessInvalidations:1883`
- 「Nanite と非 Nanite の描画差分」 → `RenderVirtualShadowMapsNanite:3789` / `RenderVirtualShadowMapsNonNanite:3956`
- 「SMRT でサンプル数を変える」 → `r.Shadow.Virtual.SMRT.RayCount*` + `VirtualShadowMapProjection.cpp`
- 「Clipmap レベル数の決定」 → `FVirtualShadowMapClipmap` + ビュー距離
