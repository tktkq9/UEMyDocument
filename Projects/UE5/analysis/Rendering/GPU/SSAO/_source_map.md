# GPU SSAO ソースマップ

- 対象: SSAO / GTAO GPU シェーダー + Lumen Short Range AO
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_ssao_gpu_overview]]

UE5 標準は GTAO（Ground-Truth AO）。Setup → HorizonSearch → Integrate → Spatial → Temporal → Upsample。
Lumen 有効時は Screen Space Short Range AO（Bent Normal 同時計算）が代替。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/PostProcessAmbientOcclusion.usf` |
| Lumen AO | `Engine/Shaders/Private/Lumen/LumenScreenSpaceBentNormal.usf` |
| CPU | `Renderer/Private/PostProcess/PostProcessAmbientOcclusion.cpp` |

---

## ファイル → シェーダー対応

### GTAO パス

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `PostProcessAmbientOcclusion.usf` | `MainSetupPS()` | `FRCPassPostProcessAmbientOcclusionSetup::Process()` | 半解像度 Depth + Normal 準備 | [[detail_ssao]] |
| `PostProcessAmbientOcclusion.usf` | `HorizonSearchCS()` | `RenderGTAO()` / `FGTAOContext` | Horizon Angle 検索 | 同 |
| `PostProcessAmbientOcclusion.usf` | `GTAOInnerIntegrateCS()` | 同上 | Horizon → AO + Bent Normal 積分 | 同 |
| `PostProcessAmbientOcclusion.usf` | `GTAOCombinedCS()` | 同上（EAsyncCombinedSpatial 時） | HorizonSearch + Integrate 統合 | 同 |
| `PostProcessAmbientOcclusion.usf` | `GTAOSpatialFilterCS()` | 同上 | 法線・深度 aware 空間フィルタ | 同 |
| `PostProcessAmbientOcclusion.usf` | `GTAOTemporalFilterCS()` | 同上 | Temporal Reprojection | 同 |
| `PostProcessAmbientOcclusion.usf` | `GTAOUpsamplePS()` | 同上 | バイラテラルアップサンプル | 同 |

### Lumen Short Range AO

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `Lumen/LumenScreenSpaceBentNormal.usf` | `ScreenSpaceShortRangeAOCS()` | `RenderLumenScreenSpaceBentNormal()` | 短距離 AO + Bent Normal | [[detail_lumen_ao]] |

---

## GPU データフロー

```
[Non-Lumen: GTAO]                   RenderGTAO() / FGTAOContext
  [1] MainSetupPS             → 半解像度 Depth + Normal
  [2] HorizonSearchCS         → 方向別 Horizon Angle
  [3] GTAOInnerIntegrateCS    → AO 値 + Bent Normal
  [4] GTAOCombinedCS          → (EAsyncCombinedSpatial: 2+3 統合)
  [5] GTAOSpatialFilterCS     → 空間フィルタ
  [6] GTAOTemporalFilterCS    → Temporal 蓄積
  [7] GTAOUpsamplePS          → 全解像度 AO

[Lumen: Short Range AO]             RenderLumenScreenSpaceBentNormal
  [8] LumenScreenSpaceBentNormal.usf:ScreenSpaceShortRangeAOCS
      → Bent Normal テクスチャ（Lumen DiffuseIndirectComposite で使用）
```

---

## 主要パーミュテーション / EGTAOType

| enum | 用途 |
|------|------|
| `EGTAOType::EOff` | SSAO モード（レガシー） |
| `EGTAOType::EAsyncCombinedSpatial` | HorizonSearch + Integrate + Spatial を統合 |
| `EGTAOType::EAsyncHorizonSearch` | HorizonSearch を Async Compute |
| `EGTAOType::ENonAsync` | 同期実行 |

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_ssao]] / [[detail_lumen_ao]] |
| Reference | [[ref_ssao]] / [[ref_lumen_ao]] |

---

## ue5-dive 起点

- 「GTAO 全体」 → `PostProcessAmbientOcclusion.cpp:RenderGTAO` + `FGTAOContext`
- 「Horizon Search」 → `PostProcessAmbientOcclusion.usf:HorizonSearchCS`
- 「Integrate + Bent Normal」 → `GTAOInnerIntegrateCS`
- 「Combined 版」 → `GTAOCombinedCS` + `EGTAOType::EAsyncCombinedSpatial`
- 「Lumen Short Range AO」 → `LumenScreenSpaceBentNormal.usf:ScreenSpaceShortRangeAOCS`
