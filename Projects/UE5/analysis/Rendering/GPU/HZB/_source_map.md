# GPU HZB ソースマップ

- 対象: HZB GPU シェーダー（Furthest/Closest ミップチェーン + Occlusion Test）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_hzb_gpu_overview]]

GroupShared を使った Reduce で 1 Dispatch 最大 4 Mip を同時書き出し。
FurthestHZB（Min-Reduce / Reverse-Z 前提）と ClosestHZB（Max-Reduce）を同時生成。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/HZB.usf` |
| シェーダー | `Engine/Shaders/Private/HZBOcclusion.usf` |
| CPU | `Renderer/Private/SceneHZB.cpp` / `SceneOcclusion.cpp` |

---

## ファイル → シェーダー対応

### HZB Build

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `HZB.usf` | `HZBBuildCS()` | `BuildHZB()` (`SceneHZB.cpp`) | GroupShared Reduce・1 Dispatch で最大 4 Mip | [[detail_hzb_build]] |
| `HZB.usf` | `HZBBuildPS()` | 同上（PS フォールバック） | Gather4 + SampleLevel ベース | 同 |

### Occlusion Test

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `HZBOcclusion.usf` | `HZBTestPS()` | `FHZBOcclusionTester::Submit()` → `TestHZB()` | AABB 投影 → IsVisibleHZB() | [[detail_occlusion_test]] |

---

## GPU データフロー

```
[1] HZB Build                       BuildHZB()
    HZB.usf:HZBBuildCS（4 Mip/Dispatch）
      → FurthestHZB (R16F ミップチェーン)
      → ClosestHZB  (R16F ミップチェーン)

[2] Occlusion Test                  FHZBOcclusionTester::Submit
    HZBOcclusion.usf:HZBTestPS
      → AABB 投影 → FurthestHZB 適切 Mip と比較
      → OutColor (1=可視 / 0=不可視)
      → Readback → PrimitiveVisibilityMap
```

---

## 主要パーミュテーション

| マクロ | 用途 |
|--------|------|
| `DIM_MIP_LEVEL_COUNT=4` | 1 Dispatch 4 Mip 書き出し |
| `HZB_TYPE_FURTHEST` | Min Reduce（Reverse-Z 標準） |
| `HZB_TYPE_CLOSEST` | Max Reduce（GPU Driven カリング用） |

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_hzb_build]] / [[detail_occlusion_test]] |
| Reference | [[ref_hzb_build]] / [[ref_occlusion_test]] |

---

## ue5-dive 起点

- 「HZB 構築 CS」 → `HZB.usf:HZBBuildCS`
- 「HZB 構築 PS フォールバック」 → `HZB.usf:HZBBuildPS`
- 「Occlusion Test」 → `HZBOcclusion.usf:HZBTestPS` + `FHZBOcclusionTester::Submit`
- 「Furthest/Closest 使い分け」 → `HZB_TYPE_*` パーミュテーション
