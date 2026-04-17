# HZB GPU 処理概要

- グループ: HZB GPU
- CPU 概要: [[21_hzb_overview]]
- CPU 詳細: [[a_hzb_build]] | [[b_hzb_occlusion]]

---

## HZB GPU パイプライン実行順

HZB（Hierarchical Z-Buffer）は **深度バッファのミップチェーン**を生成し、  
SSR / Lumen / Nanite の遮蔽判定・Occlusion Query フィードバックに利用する。

```
[HZB 構築]
[1]  HZB Build（Mip Chain 生成）
     ├─ HZBBuildCS（HZB.usf）… GroupShared を使ったバッチ Reduce（最大 4 Mip/Dispatch）
     └─ HZBBuildPS（HZB.usf）… PS フォールバック（Gather4 + SampleLevel）

[Occlusion Test]
[2]  HZB Occlusion Test
     └─ HZBTestPS（HZBOcclusion.usf）… AABB を投影して IsVisibleHZB() でサンプル比較
```

---

## 各ステップの詳細

### [1] HZB Build

| 項目 | 内容 |
|-----|------|
| **概要** | SceneDepth から FurthestHZB（Min-Reduce / Reverse-Z）と ClosestHZB（Max-Reduce）を同時生成。1 Dispatch で最大 4 Mip を書き出す |
| **CPU 関数** | `BuildHZB()` (`SceneHZB.cpp`) |
| **シェーダー** | CS: `HZB.usf#HZBBuildCS()`; PS: `HZB.usf#HZBBuildPS()` |
| **出力** | `FurthestHZB`（R16F ミップチェーン）/ `ClosestHZB`（R16F ミップチェーン）|
| **CPU 詳細** | [[a_hzb_build]] |
| **GPU シェーダー詳細** | [[detail_hzb_build]] / [[ref_hzb_build]] |

---

### [2] HZB Occlusion Test

| 項目 | 内容 |
|-----|------|
| **概要** | プリミティブの AABB を投影して FurthestHZB の適切な Mip と比較。可視なら OutColor = 1 |
| **CPU 関数** | `FHZBOcclusionTester::Submit()` → `TestHZB()` |
| **シェーダー** | PS: `HZBOcclusion.usf#HZBTestPS()` |
| **出力** | `OutColor`（1 = 可視 / 0 = 不可視）→ Readback → `PrimitiveVisibilityMap` |
| **CPU 詳細** | [[b_hzb_occlusion]] |
| **GPU シェーダー詳細** | [[detail_occlusion_test]] / [[ref_occlusion_test]] |

---

## シェーダー別 CPU 対応一覧

| シェーダーファイル | 主要エントリポイント | CPU 関数 | グループ |
|----------------|----------------|---------|---------|
| `HZB.usf` | `HZBBuildCS()` | `BuildHZB()` | [[a_HZBBuild]] |
| `HZB.usf` | `HZBBuildPS()` | `BuildHZB()` (PS fallback) | [[a_HZBBuild]] |
| `HZBOcclusion.usf` | `HZBTestPS()` | `FHZBOcclusionTester::Submit()` | [[b_OcclusionTest]] |
