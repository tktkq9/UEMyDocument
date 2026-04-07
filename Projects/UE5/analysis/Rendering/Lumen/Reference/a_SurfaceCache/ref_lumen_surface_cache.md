# リファレンス：LumenSurfaceCache.cpp

- グループ: a - Surface Cache
- 上位: [[a_lumen_surface_cache]]
- 関連: [[ref_lumen_scene_data]] | [[ref_lumen_mesh_cards]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSurfaceCache.cpp`

---

## 概要

Surface Cache の**物理ページアロケーション・アトラス管理・更新スケジューリング**を実装するファイル。  
`FLumenSurfaceCacheAllocator` を使って Card のアトラス領域を割り当て・解放し、  
どの Card を今フレームで更新するかを決定する。

---

## 主要処理フロー

```
毎フレーム:
  1. ResLevelAllocation（解像度決定）
     ├─ カメラ距離 → DesiredLockedResLevel を計算
     └─ アトラス空き容量 → 実際の ResLevel を決定（足りなければ格下げ）

  2. ページアロケーション
     ├─ FLumenSurfaceCacheAllocator::Allocate() で物理ページを確保
     └─ FLumenPageTableEntry に PhysicalPageCoord / PhysicalAtlasRect を書き込む

  3. 更新対象の選定（CardCaptureRequest リスト作成）
     ├─ 新規に割り当てられたページ → 優先してキャプチャ
     ├─ フィードバックで参照頻度が高いページ → 高解像度割り当て
     └─ r.LumenScene.SurfaceCache.CardCaptureRefreshFraction 分の既存ページ再キャプチャ

  4. ページ解放
     └─ 参照されなくなった Card のページを FLumenSurfaceCacheAllocator::Free() で解放
```

---

## 関連するアロケータの仕組み（FLumenSurfaceCacheAllocator 詳細）

物理ページ (128×128 テクセル) を **ビン方式** でサブアロケーションする。

```
物理ページ (128×128)
  ├─ 全体を 1 要素として使う（大きな Card 用）
  └─ サブアロケーションモード（小さな Card 用）
       ├─ 8×8   → 16×16 = 256 個詰め込める
       ├─ 8×16  → 16×8  = 128 個
       ├─ 16×16 → 8×8   = 64 個
       └─ ... （最大 64 種類の FPageBin）

FPageBinLookup[log2(W)][log2(H)] → FPageBin へのインデックス（O(1) 検索）
```

---

## 主要関数（LumenSurfaceCache.cpp 内）

| 関数名 | 役割 |
|-------|------|
| `UpdateSurfaceCacheAllocationState(...)` | 全 Card の解像度・ページ割り当て状態を更新 |
| `BuildCardUpdateList(...)` | 今フレームで更新すべき Card ページのリストを作成 |
| `AllocateSurfaceCacheAtlas(...)` | アトラステクスチャの生成・リサイズ |
| `FreeSurfaceCacheAllocation(Card, Allocator)` | 指定 Card の物理ページを解放 |
| `GetSurfaceCacheCompression()` | 圧縮方式を返す（`ESurfaceCacheCompression`）|

---

## 主要 CVar

```
r.LumenScene.SurfaceCache.AtlasSize = 4096
    → アトラスの一辺のサイズ（px）

r.LumenScene.SurfaceCache.CardCapturesPerFrame = 300
    → 1 フレームあたり最大キャプチャ Card 数

r.LumenScene.SurfaceCache.CardCaptureFactor = 64
    → キャプチャテクセル上限 = 総テクセル数 / Factor

r.LumenScene.SurfaceCache.CardCaptureRefreshFraction = 0.125
    → 既存ページ再キャプチャに使えるバジェット割合

r.LumenScene.SurfaceCache.RemovesPerFrame = 512
    → 1 フレームあたり最大 Card 削除数

r.LumenScene.SurfaceCache.Freeze = 0
    → Surface Cache 更新を凍結（デバッグ用）

r.LumenScene.SurfaceCache.Reset = 0
    → Surface Cache を全リセット（デバッグ用）

r.LumenScene.SurfaceCache.ResetEveryNthFrame = 0
    → N フレームごとに全リセット（デバッグ用）

r.LumenScene.SurfaceCache.CardFixedDebugResolution = -1
    → 全 Card を固定解像度にする（デバッグ用、-1 = 無効）
```
