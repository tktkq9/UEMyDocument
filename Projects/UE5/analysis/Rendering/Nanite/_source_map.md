# Nanite ソースマップ

- 対象: 仮想化ジオメトリ（Cull & Raster + Shading）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[03_nanite_overview]]

Instance→Cluster→Triangle の GPU Driven カリング + HW/SW ハイブリッドラスタ。
結果を `VisBuffer64`（Depth40 + MaterialDepth24）に書き、マテリアルビン別に遅延シェーディング。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/Nanite/` |
| シェーダー | `Engine/Shaders/Private/Nanite/*.usf` |
| 呼び出し元 | `Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp`, `BasePassRendering.cpp` |

---

## ファイル → クラス対応

### エントリ・共通

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `Nanite.h/.cpp` | `FSharedContext`, `FRasterResults`, `EPipeline`, `ERasterScheduling` | エントリ・パイプライン種別・ラスタスケジューリング | — |
| `NaniteShared.h/.cpp` | `Nanite::FPackedView`, `FPackedViewArray`, `FGlobalResources` | ビュー・グローバルリソース・基底シェーダー | — |

### Cull & Raster

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `NaniteCullRaster.h` | `IRenderer::Create():181`, `DrawGeometry():197`, `ExtractResults():250` | カリング・ラスタ Factory + 実行 API | [[a_nanite_cull_raster]] |
| `NaniteCullRaster.cpp` | 2パスオクルージョンカリング・HW/SW 切替 | Instance→Cluster→Triangle GPU カリング | 同 |

### Materials & Shading

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `NaniteMaterials.h/.cpp` | マテリアルスロット・ビン分類定義 | EMeshPass → Material ビン | [[b_nanite_materials_shading]] |
| `NaniteMaterialsSceneExtension.h/.cpp` | マテリアルバッファ GPU 管理 | 非同期アップロード | 同 |
| `NaniteShading.h/.cpp` | `BuildShadingCommands():49`, `DispatchBasePass():1178`, `ShadeBinning():1443`, `EmitGBuffer` | ビン構築・CS ディスパッチ・GBuffer 書き込み | 同 |
| `NaniteDrawList.h/.cpp` | ドロー命令リスト | ビン登録・パイプライン遅延構築 | — |

### Composition

| ファイル | 主要関数 | 役割 |
|---------|--------|------|
| `NaniteComposition.h/.cpp` | `EmitDepth`, `EmitCustomDepth`, `EmitStencil` | SceneDepth / CustomDepth / Stencil 最終出力 |

### Visibility / Ownership

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `NaniteVisibility.h/.cpp` | 可視性クエリ・並列タスク | 可視性結果の抽出 | [[c_nanite_visibility]] |
| `NaniteOwnershipVisibilitySceneExtension.h/.cpp` | bOwnerNoSee / bOnlyOwnerSee の GPU フィルタ | オーナーシップ別可視性 | 同 |

### Ray Tracing / StreamOut

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `NaniteRayTracing.h/.cpp` | `FRayTracingManager` | BLAS 構築キュー | [[d_nanite_ray_tracing]] |
| `NaniteStreamOut.h/.cpp` | GPU→CPU メッシュデータ抽出 | RT 用 | 同 |

### Tessellation / Voxel

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `TessellationTable.cpp` | テッセレーションテーブル事前生成 | マイクロポリゴン | [[e_nanite_tess_voxel]] |
| `Voxel.h/.cpp` | ボクセルブリックデバッグ描画 | 同 | 同 |

### Debug / Editor

| ファイル | 役割 | 参照 |
|---------|------|------|
| `NaniteVisualize.h/.cpp` | デバッグビュー・ピッキング | [[f_nanite_debug_editor]] |
| `NaniteEditor.h/.cpp` | ヒットプロキシ・選択オーバーレイ | 同 |
| `NaniteFeedback.h/.cpp` | オーバーフロー検出・警告 | 同 |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─ RenderNanite()                         DeferredShadingRenderer.cpp:1364
  │    ├─ Nanite::InitRasterContext            :1451
  │    │   └─ VisBuffer64 / DbgBuffer64 / ShadingMaskBuffer 確保
  │    ├─ Nanite::IRenderer::Create            :1676   NaniteCullRaster.h:181
  │    ├─ NaniteRenderer->DrawGeometry         :1689   NaniteCullRaster.h:197
  │    │   └─ Instance → Cluster → Triangle GPU カリング
  │    │       → HW ラスタ ∥ SW ラスタ（非同期コンピュート）
  │    │       → VisBuffer64 書き込み
  │    └─ NaniteRenderer->ExtractResults       :1696
  │
  ├─ Nanite::BuildShadingCommands             :2599   NaniteShading.h:49
  │    └─ ENaniteMeshPass::BasePass / LumenCardCapture の IndirectDispatch 引数構築
  │
  └─ RenderBasePass → RenderBasePassInternal   BasePassRendering.cpp:1449
       └─ RenderNaniteBasePass lambda          :1516
            └─ Nanite::DispatchBasePass        :1543 → NaniteShading.cpp:1178
                 └─ ShadeBinning               NaniteShading.cpp:1443
                    ├─ ClassifyPixels
                    └─ BuildShadingBinArgs
                 → IndirectDispatch × ビン数 → GBuffer
```

---

## EPipeline（`Nanite.h`）

| enum | 用途 |
|------|------|
| `Primary` | 通常 BasePass |
| `Shadows` | Shadow Depth（VSM 含む） |
| `Lumen` | Lumen Surface Cache キャプチャ |
| `Editor` | エディタ選択オーバーレイ |
| `HitProxy` | ヒットプロキシ |

## ERasterScheduling

| enum | 意味 |
|------|------|
| `HardwareOnly` | HW ラスタのみ |
| `HardwareThenSoftware` | HW → SW 同期 |
| `HardwareAndSoftwareOverlap` | HW ∥ SW 非同期コンピュート |

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.Nanite.AsyncRasterization` | `NaniteCullRaster.cpp` |
| `r.Nanite.ComputeRasterization` | `NaniteCullRaster.cpp` |
| `r.Nanite.ProgrammableRaster` | `NaniteCullRaster.cpp` |
| `r.Nanite.Tessellation` | `NaniteCullRaster.cpp` |
| `r.Nanite.MaxPixelsPerEdge` | `NaniteCullRaster.cpp` |
| `r.Nanite.MinPixelsPerEdgeHW` | `NaniteCullRaster.cpp` |
| `r.Nanite.DicingRate` | `NaniteCullRaster.cpp` |
| `r.Nanite.MeshShaderRasterization` | `NaniteCullRaster.cpp` |
| `r.Nanite.ResummarizeHTile` | `NaniteComposition.cpp` |
| `r.Nanite.Shadows` | `NaniteShadowDepthRendering.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Details | [[a_nanite_cull_raster]] | Cull & Raster |
| Details | [[b_nanite_materials_shading]] | Materials & Shading |
| Details | [[c_nanite_visibility]] | Visibility / Ownership |
| Details | [[d_nanite_ray_tracing]] | BLAS / StreamOut |
| Details | [[e_nanite_tess_voxel]] | Tessellation / Voxel |
| Details | [[f_nanite_debug_editor]] | Debug / Editor |

> Reference ドキュメントが存在する場合は `Reference/` サブフォルダを参照。

---

## ue5-dive 起点

- 「Nanite のエントリ」 → `DeferredShadingRenderer.cpp:RenderNanite:1364`
- 「2パスカリングの実体」 → `NaniteCullRaster.cpp:DrawGeometry` + 前フレーム Depth オクルージョン
- 「HW/SW ラスタの切替閾値」 → `r.Nanite.MinPixelsPerEdgeHW`
- 「マテリアルビン分類」 → `NaniteShading.cpp:ShadeBinning:1443`
- 「Nanite → GBuffer 書き込み」 → `NaniteShading.cpp:DispatchBasePass:1178`
- 「Nanite Shadow」 → `EPipeline::Shadows` + `NaniteShadowDepthRendering.cpp`
- 「Lumen Card キャプチャ」 → `EPipeline::Lumen` + `NaniteShading.cpp:BuildShadingCommands`
