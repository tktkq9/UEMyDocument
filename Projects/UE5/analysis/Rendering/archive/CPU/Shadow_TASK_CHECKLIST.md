# Shadow Rendering ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\`  
**注意**: Virtual Shadow Maps（VSM）は別フォルダ `VirtualShadowMaps/` を参照。  
本フォルダは非 VSM の従来シャドウマップ（Cascaded SM / Spot / Point）を対象とする。

---

## 概要ファイル

- [x] `17_shadow_overview.md` — Shadow システム全体像・従来 SM と VSM の使い分け・FProjectedShadowInfo の生存期間

---

## Phase 1: Details（3本）

- [x] `Details/a_shadow_setup.md`      — `FProjectedShadowInfo` 生成・種別分類（Whole Scene / PerObject / Spot 等）
- [x] `Details/b_shadow_depth.md`      — Shadow Depth Map 描画（非 Nanite / Nanite 経由）・アトラスパッキング
- [x] `Details/c_shadow_projection.md` — Shadow Projection（PCF / PCSS）・Shadow Mask の合成

---

## Phase 2: Reference（3本）

- [x] `Reference/ref_projected_shadow_info.md` — `FProjectedShadowInfo` 全メンバ・`FShadowCascadeSettings`
- [x] `Reference/ref_shadow_rendering.md`       — `FShadowDepthVS` / `FShadowDepthPS` / `FShadowDepthMeshProcessor`
- [x] `Reference/ref_shadow_projection.md`      — `FShadowProjectionPS` / `FShadowPCF` / `TShadowProjectionPS` テンプレート

---

## Phase 3: コード実行フロー追加

- [x] `17_shadow_overview.md`             — `RenderShadowDepthMaps()` → 種別分類 → Depth 描画 → Projection フロー追加
- [x] `Details/a_shadow_setup.md`         — `SetupMeshDrawCommandsForShadowDepth()` → FProjectedShadowInfo 生成フロー
- [x] `Details/b_shadow_depth.md`         — アトラス確保 → Nanite / 非Nanite ディスパッチ → 深度書き込みフロー
- [x] `Details/c_shadow_projection.md`    — Shadow Mask 生成 → PCF → LightAttenuation テクスチャ合成フロー

---

## 進捗サマリ

| フェーズ | 合計 | 完了 | 残り |
|---------|------|------|------|
| 概要 | 1 | 1 | 0 |
| Phase 1: Details | 3 | 3 | 0 |
| Phase 2: Reference | 3 | 3 | 0 |
| Phase 3: Flow | 4 | 4 | 0 |
| **合計** | **11** | **11** | **0** |

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `ShadowRendering.h/.cpp` | FProjectedShadowInfo・Shadow Projection パス |
| `ShadowSetup.cpp` | シャドウ種別分類・アトラスパッキング |
| `ShadowDepthRendering.h/.cpp` | Shadow Depth Map 描画・MeshPassProcessor |
| `ShadowProjectionPixelShader.h` | FShadowProjectionPS テンプレート群 |
