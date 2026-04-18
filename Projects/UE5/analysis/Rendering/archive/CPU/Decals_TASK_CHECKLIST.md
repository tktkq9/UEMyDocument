# Deferred Decals ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\`  
**GPU 対応**: [[GPU/Decals/TASK_CHECKLIST]]

---

## 概要ファイル

- [x] `15_decals_overview.md` — Deferred Decals の全体フロー・DBuffer 方式・ブレンドモード一覧

---

## Phase 1: Details（2本）

- [x] `Details/a_deferred_decal.md` — `RenderDeferredDecals()` フロー・BlendMode 別パス分離・ステンシル利用
- [x] `Details/b_dbuffer.md`        — DBuffer デカール（Pre-GBuffer 合成）・`FDBufferTextures` 生成・BasePass との連携

---

## Phase 2: Reference（2本）

- [x] `Reference/ref_deferred_decal.md` — `FDeferredDecalPS` / `FDeferredDecalRenderTargetManager` / `FTransientDecalRenderData`
- [x] `Reference/ref_dbuffer.md`        — `FDBufferTextures` / `FDBufferData` / `ApplyDBufferDecal` バインドフロー

---

## Phase 3: コード実行フロー追加

- [x] `15_decals_overview.md`        — `RenderDeferredDecals()` → BlendMode 分岐 → デカール投影フロー追加
- [x] `Details/a_deferred_decal.md`  — Stencil Mask → Sort → Batch ごとのフロー
- [x] `Details/b_dbuffer.md`         — DBuffer テクスチャ生成 → マテリアル書き込み → BasePass での ApplyDBuffer フロー

---

## 進捗サマリ

| フェーズ | 合計 | 完了 | 残り |
|---------|------|------|------|
| 概要 | 1 | 1 | 0 |
| Phase 1: Details | 2 | 2 | 0 |
| Phase 2: Reference | 2 | 2 | 0 |
| Phase 3: Flow | 3 | 3 | 0 |
| **合計** | **8** | **8** | **0** |

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `DeferredDecal.h/.cpp` | Deferred Decals レンダリング・DBuffer 生成 |
| `DecalRenderingShared.h` | FTransientDecalRenderData / ブレンドモード判定 |
