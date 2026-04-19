# HZB ソースマップ

- 対象: 階層化 Z バッファ（FurthestHZB / ClosestHZB）の構築と利用
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[21_hzb_overview]]

`BuildHZB()` が SceneDepth から Min/Max Reduce のミップチェーンを生成し、
Occlusion Culling / SSR / Lumen / Nanite Persistent Cull / Shadow Cull で共用する。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/` |
| HZB 本体 | `HZB.h/.cpp` |
| Occlusion クエリ | `SceneOcclusion.h/.cpp` |
| シェーダー | `Engine/Shaders/Private/HZBBuild.usf`, `HZBOcclusion.usf` |
| 呼び出し元 | `DeferredShadingRenderer.cpp`（InitViews→Occlusion） |

---

## ファイル → クラス対応

### HZB 構築

| ファイル | 主要関数 / 構造体 | 役割 | 参照 |
|---------|----------------|------|------|
| `HZB.h/.cpp` | `BuildHZB()`, `FHZBParameters`, `EHZBType` | SceneDepth → ミップ付き HZB（Closest/Furthest）| [[Reference/ref_hzb_resources]] |
| `HZB.h` | `FSceneHZB` | ビューに保持される HZB テクスチャ群 | 同 |

### Occlusion Culling

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `SceneOcclusion.h/.cpp` | `FHZBOcclusionTester`, `RenderOcclusion()`, `FViewOcclusionQueries`, `FOcclusionQueryVS` | HZB AABB テスト・Readback → PrimitiveVisibilityMap | [[Reference/ref_hzb_occlusion]] |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─[InitViews]
  │   RenderOcclusion                           SceneOcclusion.cpp
  │     └─ FHZBOcclusionTester: 前フレーム HZB で AABB テスト
  │         → Readback Buffer → 翌フレームの PrimitiveVisibilityMap
  │
  ├─[GBuffer / BasePass 後]
  │   BuildHZB(GraphBuilder, SceneDepth, ...)   HZB.cpp
  │     ├─ Mip0: SceneDepth を 1/2 にダウンサンプル（Min/Max）
  │     ├─ Mip1-N: 前ミップから 2x2 Reduce（CS）
  │     └─ FViewInfo::HZB に Furthest / Closest を保存
  │
  └─[各システムで利用]
      SSR / Lumen Screen Probe / Nanite Persistent Cull / Shadow Cull
```

---

## EHZBType（HZB.h）

| enum | 用途 |
|------|------|
| `Dummy` | 未使用スロット |
| `ClosestHZB` | 最小深度（Contact Shadow / SSGI） |
| `FurthestHZB` | 最大深度（Occlusion / SSR / Lumen / Nanite） |
| `All` | 両方構築 |

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.HZBOcclusion` | `SceneOcclusion.cpp` |
| `r.HZB.BuildMipCount` | `HZB.cpp` |
| `r.HZB.DownsampleCS` | `HZB.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_hzb_resources]] | FHZBParameters / FSceneHZB |
| Reference | [[Reference/ref_hzb_occlusion]] | FHZBOcclusionTester / OcclusionQueryVS |

---

## ue5-dive 起点

- 「HZB の構築」 → `HZB.cpp:BuildHZB`
- 「Furthest/Closest の切替」 → `EHZBType` + `FHZBParameters::bIsFurthestHZBValid/bIsClosestHZBValid`
- 「Occlusion Culling の実体」 → `SceneOcclusion.cpp:RenderOcclusion` + `FHZBOcclusionTester`
- 「1フレーム遅延の理由」 → Readback バッファを CPU が翌フレームで参照
- 「SSR / Lumen で HZB を使う場所」 → `FHZBParameters` を受け取るシェーダー（HZBTexture / ClosestHZBTexture）
