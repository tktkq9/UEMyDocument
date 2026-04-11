# Nanite リファレンスドキュメント作成 タスクチェックリスト

作成目標: `Reference/` フォルダ以下に各ソースファイルのクラス・関数リファレンスを作成し、
対応する `Details/` 記事にリンクを追加する。

---

## フォルダ構成（完成形）

```
Rendering/Nanite/
├── 03_nanite_overview.md
├── Details/
│   ├── a_nanite_cull_raster.md       ← リンク追加済み？ [x]
│   ├── b_nanite_materials_shading.md ← リンク追加済み？ [x]
│   ├── c_nanite_visibility.md        ← リンク追加済み？ [x]
│   ├── d_nanite_ray_tracing.md       ← リンク追加済み？ [x]
│   ├── e_nanite_tess_voxel.md        ← リンク追加済み？ [x]
│   └── f_nanite_debug_editor.md      ← リンク追加済み？ [x]
├── Reference/
│   ├── Common/
│   ├── a_CullRaster/
│   ├── b_MaterialsShading/
│   ├── c_Visibility/
│   ├── d_RayTracing/
│   ├── e_TessVoxel/
│   └── f_DebugEditor/
└── TASK_CHECKLIST.md（このファイル）
```

---

## グループ共通（Common）— 2本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| C-1 | `Reference/Common/ref_nanite_core.md` | `Nanite.h/.cpp` | [x] 完了 |
| C-2 | `Reference/Common/ref_nanite_shared.md` | `NaniteShared.h/.cpp` | [x] 完了 |

---

## グループ a：CullRaster — 2本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| A-1 | `Reference/a_CullRaster/ref_nanite_cull_raster.md` | `NaniteCullRaster.h/.cpp` | [x] 完了 |
| A-2 | `Reference/a_CullRaster/ref_nanite_composition.md` | `NaniteComposition.h/.cpp` | [x] 完了 |

---

## グループ b：Materials & Shading — 4本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| B-1 | `Reference/b_MaterialsShading/ref_nanite_materials.md` | `NaniteMaterials.h/.cpp` | [x] 完了 |
| B-2 | `Reference/b_MaterialsShading/ref_nanite_materials_scene_ext.md` | `NaniteMaterialsSceneExtension.h/.cpp` | [x] 完了 |
| B-3 | `Reference/b_MaterialsShading/ref_nanite_shading.md` | `NaniteShading.h/.cpp` | [x] 完了 |
| B-4 | `Reference/b_MaterialsShading/ref_nanite_draw_list.md` | `NaniteDrawList.h/.cpp` | [x] 完了 |

---

## グループ c：Visibility — 2本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| C-1 | `Reference/c_Visibility/ref_nanite_visibility.md` | `NaniteVisibility.h/.cpp` | [x] 完了 |
| C-2 | `Reference/c_Visibility/ref_nanite_ownership_visibility.md` | `NaniteOwnershipVisibilitySceneExtension.h/.cpp` | [x] 完了 |

---

## グループ d：Ray Tracing — 2本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| D-1 | `Reference/d_RayTracing/ref_nanite_ray_tracing.md` | `NaniteRayTracing.h/.cpp` | [x] 完了 |
| D-2 | `Reference/d_RayTracing/ref_nanite_stream_out.md` | `NaniteStreamOut.h/.cpp` | [x] 完了 |

---

## グループ e：Tessellation & Voxel — 2本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| E-1 | `Reference/e_TessVoxel/ref_nanite_tessellation_table.md` | `TessellationTable.cpp` | [x] 完了 |
| E-2 | `Reference/e_TessVoxel/ref_nanite_voxel.md` | `Voxel.h/.cpp` | [x] 完了 |

---

## グループ f：Debug & Editor — 3本

| # | ファイル名 | 対象ソース | 状態 |
|---|-----------|-----------|------|
| F-1 | `Reference/f_DebugEditor/ref_nanite_visualize.md` | `NaniteVisualize.h/.cpp` | [x] 完了 |
| F-2 | `Reference/f_DebugEditor/ref_nanite_editor.md` | `NaniteEditor.h/.cpp` | [x] 完了 |
| F-3 | `Reference/f_DebugEditor/ref_nanite_feedback.md` | `NaniteFeedback.h/.cpp` | [x] 完了 |

---

## 進捗サマリ

| グループ | 合計 | 完了 | 残り |
|---------|------|------|------|
| Common | 2 | 2 | 0 |
| a: CullRaster | 2 | 2 | 0 |
| b: Materials & Shading | 4 | 4 | 0 |
| c: Visibility | 2 | 2 | 0 |
| d: Ray Tracing | 2 | 2 | 0 |
| e: Tessellation & Voxel | 2 | 2 | 0 |
| f: Debug & Editor | 3 | 3 | 0 |
| **合計** | **17** | **17** | **0** |

---

## Phase 3 — コード実行フロー追加・Reference 強化

### Overview
- [x] 03_nanite_overview: RenderNanite → DrawGeometry → DispatchBasePass 全体フロー追加

### Details（`## コード実行フロー` 追加）
- [x] a_nanite_cull_raster    : InitRasterContext → IRenderer::Create → DrawGeometry（2パス） → ExtractResults
- [x] b_nanite_materials_shading: BuildShadingCommands → ShadeBinning → DispatchBasePass フロー
- [x] c_nanite_visibility     : BeginVisibilityQuery → PerformNaniteVisibility（非同期タスク）→ GetVisibilityResults
- [x] d_nanite_ray_tracing    : Add/Remove → UpdateStreaming → ProcessBuildRequests → BLAS 構築
- [x] e_nanite_tess_voxel     : テッセレーション・ボクセル描画 フロー
- [x] f_nanite_debug_editor   : AddVisualizationPasses / DrawHitProxies / FFeedbackManager フロー

### Reference（メンバ変数テーブル化・`> [!note]-` 追加）
- [x] ref_nanite_core         : FSharedContext / FRasterContext / FRasterResults テーブル化・3 callout
- [x] ref_nanite_shared       : FPackedView テーブル・3 callout（GPU アップロード / グローバルシェーダー / 永続リソース）
- [x] ref_nanite_cull_raster  : 3 callout（2パス遅延 / DepthOnly / ShadingMaskBuffer）
- [x] ref_nanite_shading      : FShadeBinning / FNaniteShadingPipeline テーブル化・関数テーブル行番号追加・3 callout

---

## 作業ルール

1. 1セッションで **同グループをまとめて**依頼するのが効率的
2. リファレンス記事には以下を含める：
   - クラス一覧（継承・役割の一言説明）
   - 主要関数一覧（引数・戻り値・役割）
   - 主要マクロ（`SHADER_PARAMETER`系）
   - 関連ファイルリンク
3. Detail記事の末尾に `## 関連リファレンス` セクションを追加してリンクを張る
4. 完了したタスクは `[x]` に更新してコミット
