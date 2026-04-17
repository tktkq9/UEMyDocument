# Screen Space AO GPU 処理概要

- グループ: SSAO GPU
- CPU 概要: [[22_ssao_overview]]
- CPU 詳細: [[a_ssao]] | [[b_composition_lighting]]

---

## SSAO GPU パイプライン実行順

UE5 は **GTAO（Ground-Truth Ambient Occlusion）** を標準の SSAO 実装として採用。  
Lumen 有効時は **Screen Space Short Range AO** が使われる。

```
[SSAO / GTAO パス（非 Lumen 時）]
[1]  Setup Pass
     └─ MainSetupPS … SceneDepth + GBuffer Normal → 半解像度に縮小

[2]  Horizon Search（GTAO 固有）
     └─ HorizonSearchCS … 各ピクセルから複数方向の Horizon Angle を検索

[3]  Integrate（GTAO 固有）または SSAO Sampling
     ├─ GTAOInnerIntegrateCS … Horizon Angle → AO 値に積分（Bent Normal 計算）
     └─ MainPS / MainCS … SSAO 半球サンプリング（従来 SSAO モード）

[4]  Combined（HorizonSearch + Integrate 一体版）
     └─ GTAOCombinedCS … EGTAOType::EAsyncCombinedSpatial 時の統合パス

[5]  Spatial Filter（ブラー）
     └─ GTAOSpatialFilterCS … 法線・深度aware な空間フィルタ

[6]  Temporal Filter
     └─ GTAOTemporalFilterCS … Temporal Reprojection による蓄積

[7]  Upsample（半解像度 → 全解像度）
     └─ GTAOUpsamplePS … バイラテラルアップサンプル

[Lumen AO パス（Lumen 有効時）]
[8]  Screen Space Short Range AO
     └─ ScreenSpaceShortRangeAOCS（LumenScreenSpaceBentNormal.usf）… 短距離 AO + Bent Normal
```

---

## 各ステップの詳細

### [1] Setup Pass

| 項目 | 内容 |
|-----|------|
| **概要** | SceneDepth と GBuffer Normal を半解像度に縮小してサンプリング負荷を下げる |
| **CPU 関数** | `FRCPassPostProcessAmbientOcclusionSetup::Process()` |
| **シェーダー** | PS: `PostProcessAmbientOcclusion.usf#MainSetupPS()` |
| **出力** | 半解像度 Depth + Normal テクスチャ |
| **GPU シェーダー詳細** | [[detail_ssao]] / [[ref_ssao]] |

---

### [2〜4] GTAO Horizon Search + Integrate

| 項目 | 内容 |
|-----|------|
| **概要** | 各ピクセルから複数方向の Horizon Angle を検索し、AO値と Bent Normal を計算 |
| **CPU 関数** | `RenderGTAO()` / `FGTAOContext` (`PostProcessAmbientOcclusion.cpp`) |
| **シェーダー** | `HorizonSearchCS` / `GTAOInnerIntegrateCS` / `GTAOCombinedCS` |
| **出力** | R8（AO）または R8G8（AO + Bent Normal）|
| **GPU シェーダー詳細** | [[detail_ssao]] / [[ref_ssao]] |

---

### [5〜7] フィルタリング・アップサンプル

| 項目 | 内容 |
|-----|------|
| **概要** | 空間ブラー → テンポラル蓄積 → 全解像度にアップサンプル |
| **シェーダー** | `GTAOSpatialFilterCS` / `GTAOTemporalFilterCS` / `GTAOUpsamplePS` |
| **出力** | `SceneColor` に合成される AO テクスチャ |

---

### [8] Screen Space Short Range AO（Lumen 専用）

| 項目 | 内容 |
|-----|------|
| **概要** | Lumen の GI と連携する短距離 AO。Bent Normal も同時に計算 |
| **CPU 関数** | `RenderLumenScreenSpaceBentNormal()` |
| **シェーダー** | CS: `LumenScreenSpaceBentNormal.usf#ScreenSpaceShortRangeAOCS()` |
| **出力** | Bent Normal テクスチャ（Lumen DiffuseIndirect Composite で使用）|
| **GPU シェーダー詳細** | [[detail_lumen_ao]] / [[ref_lumen_ao]] |

---

## シェーダー別 CPU 対応一覧

| シェーダーファイル | 主要エントリポイント | CPU 関数 | グループ |
|----------------|----------------|---------|---------|
| `PostProcessAmbientOcclusion.usf` | `MainSetupPS()` | `FRCPassPostProcessAmbientOcclusionSetup` | [[a_SSAO]] |
| `PostProcessAmbientOcclusion.usf` | `HorizonSearchCS()` | `RenderGTAO()` | [[a_SSAO]] |
| `PostProcessAmbientOcclusion.usf` | `GTAOInnerIntegrateCS()` | `RenderGTAO()` | [[a_SSAO]] |
| `PostProcessAmbientOcclusion.usf` | `GTAOCombinedCS()` | `RenderGTAO()` | [[a_SSAO]] |
| `PostProcessAmbientOcclusion.usf` | `GTAOSpatialFilterCS()` | `RenderGTAO()` | [[a_SSAO]] |
| `PostProcessAmbientOcclusion.usf` | `GTAOTemporalFilterCS()` | `RenderGTAO()` | [[a_SSAO]] |
| `PostProcessAmbientOcclusion.usf` | `GTAOUpsamplePS()` | `RenderGTAO()` | [[a_SSAO]] |
| `Lumen/LumenScreenSpaceBentNormal.usf` | `ScreenSpaceShortRangeAOCS()` | `RenderLumenScreenSpaceBentNormal()` | [[b_LumenAO]] |
