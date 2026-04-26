---
name: Lumen Final Gather (ScreenProbeGather)
description: スクリーン空間プローブによる Lumen Final Gather の理論と UE 実装
type: project
---

# Lumen Final Gather - ScreenProbeGather（スクリーンプローブ Final Gather）

- 上位: [[_algorithm_index]]
- 関連: [[lumen_surface_cache]] / [[lumen_radiance_cache]] / [[lumen_sw_rt]] / [[lumen_hw_rt]] / [[lumen_restir_gather]]
- 採用システム: Lumen Diffuse GI（Software / Hardware Ray Tracing 両方の最終統合段）
- 出典:
  - **S10**: Wright / Heitz / Hillaire 2022 SIGGRAPH "Lumen: Real-time Global Illumination in Unreal Engine 5" — §6 Screen Probe Gather, §7 Importance Sampling, §8 Filtering
  - 影響元: McGuire 2017 "Stochastic Sampling for Real-Time Indirect" / Heitz 2018 "A Low-Distortion Map Between Triangle and Square"

---

## 1. 何のためのアルゴリズムか

Lumen の **Final Gather** は「目に見える各ピクセルが受け取る間接拡散光（半球積分）」を計算する段。素朴に各ピクセルから半球レイを多数飛ばすと数千億 ray/frame になりリアルタイムでは到底回らない。

### 素朴な手法の問題

| 手法 | 問題 |
|------|-----|
| ピクセルごとに 64 ray 半球サンプリング | 1080p で 100M ray/frame、毎フレーム不可 |
| Light Probe Volume（事前ベイク） | 動的光源・動的シーン破綻 |
| Per-pixel ReSTIR | reservoir 数 × 解像度で VRAM・帯域爆発（→ [[lumen_restir_gather]] が試作） |

### ScreenProbeGather の貢献

- **画面を 16×16 タイル単位にダウンサンプル**（`r.Lumen.ScreenProbeGather.DownsampleFactor=16`）→ プローブ数 1/256
- 各プローブが **8×8 octahedral 方向**（`TracingOctahedronResolution=8`）で 64 レイをトレース
- **Adaptive Probe**: 一様プローブで補間できないジオメトリ不連続部に追加プローブを生成（`NumAdaptiveProbes=8`/タイル）
- **Importance Sampling**: 過去フレームの放射輝度＋ BRDF PDF で重要方向に再分配
- **空間フィルタ + 時間フィルタ + プローブ間補間** で最終ピクセル放射輝度を再構成

---

## 2. 理論

### 2.1 球面 Octahedral マップ

各プローブの放射輝度は **オクタヘドラル投影** で 2D テクスチャに展開:

```
半球方向 ω → [0,1]² のオクタヘドラル UV
（八面体のフェイスに射影し、平面へ展開）
```

`OctahedronToUnitVector` / `UnitVectorToOctahedron` は `LumenScreenProbeCommon.ush:71` 等で双方向変換。理由: **キューブマップより歪みが均等**で、境界周辺の重複オーバーヘッドが少ない（border 1 ピクセルで足りる）。

### 2.2 半球積分の MIS 構造

各プローブの統合光は:

$$
L_{\mathrm{indirect}}(p, \omega_o) = \int_{\Omega^+} f_r(p, \omega_o, \omega_i) L_{\mathrm{trace}}(p, \omega_i) (\omega_i \cdot n) \, d\omega_i
$$

3 通りの積分手段（`r.Lumen.ScreenProbeGather.DiffuseIntegralMethod`）:

| 値 | 手段 | 内容 |
|----|------|------|
| 0 | **Pre-integrated probe** | プローブ放射輝度を SH3 / Octahedral 既積分し、ピクセル法線から評価 |
| 1 | **Importance Sample BRDF** | BRDF PDF で再サンプリング |
| 2 | **Numerical Reference** | 厳密モンテカルロ（リファレンスのみ） |

UE デフォルトは 0（`IrradianceFormat=1`: Octahedral）。ピクセルの法線・粗さに応じてタイル分類で 0 と 1 を切り替え（`FScreenProbeIntegrateCS` の Tile Classification）。

### 2.3 プローブ補間

ピクセルから 4 近傍プローブ（`ScreenProbeViewSize` の 2×2 グリッド近傍）を選び、**深度・法線重み**でステライリングなしに補間:

```
w_i = depth_weight × normal_weight × spatial_weight
```

`InterpolationDepthWeight=1.0`（葉っぱは 0.25）が距離テスト強度。プローブ深度が画面ピクセル深度から大きく外れる近傍は除外し、Adaptive Probe の追加生成トリガーになる。

---

## 3. UE 実装

### 3.1 主要パイプライン（`LumenScreenProbeGather.cpp:2576〜`）

```
RenderLumenScreenProbeGather()
├── (1) 一様プローブ配置: ScreenProbeDownsampleDepthUniformCS
├── (2) Adaptive Probe マーキング: ScreenProbeAdaptivePlacementMarkCS
├── (3) Adaptive Probe スポーン: ScreenProbeAdaptivePlacementSpawnCS
├── (4) Radiance Cache マーク・更新: LumenRadianceCache::UpdateRadianceCaches()
├── (5) BRDF PDF 生成 + Importance Sampling Rays: GenerateBRDF_PDF / GenerateImportanceSamplingRays
├── (6) プローブトレース: TraceScreenProbes()  ← SW/HW RT に分岐
├── (7) フィルタ: FilterScreenProbes()  ← Spatial Filter + SH3 化
├── (8) 統合: InterpolateAndIntegrate()  ← FScreenProbeIntegrateCS
└── (9) 履歴更新: UpdateHistoryScreenProbeGather()  ← FScreenProbeTemporalReprojectionCS
```

### 3.2 主要シェーダクラス

| クラス | usf | 役割 |
|--------|-----|------|
| `FScreenProbeDownsampleDepthUniformCS` | `LumenScreenProbeGather.usf` | 一様プローブの深度・法線ダウンサンプル |
| `FScreenProbeAdaptivePlacementMarkCS` | 同 | 補間誤差からスポーン位置をマーク |
| `FScreenProbeAdaptivePlacementSpawnCS` | 同 | 残予算（`AdaptiveProbeAllocationFraction=0.5`）内で追加プローブを生成 |
| `FScreenProbeTileClassificationBuildListsCS` | 同 | タイル分類（SimpleDiffuse / SupportImportanceSampleBRDF / SupportAll） |
| `FScreenProbeIntegrateCS` | 同 | プローブ補間 + ピクセル統合 |
| `FScreenProbeTemporalReprojectionCS` | `LumenScreenProbeGatherTemporal.usf` | History reprojection + Variance Clamp |

### 3.3 Adaptive Probe 配置

```cpp
// LumenScreenProbeGather.cpp:46-59
// 各 16x16 タイルに対し最大 8 プローブを追加生成、全体予算は一様プローブ数 × 0.5
CVarLumenScreenProbeGatherNumAdaptiveProbes = 8;
GLumenScreenProbeGatherAdaptiveProbeAllocationFraction = 0.5f;
```

ジオメトリ不連続（深度差・法線角差）が補間しきい値を超えると追加。葉っぱピクセルは `InterpolationDepthWeightForFoliage=0.25` で緩めて Adaptive Probe を抑制（葉は微小ディテール多く Adaptive が暴走するため）。

### 3.4 トレース → Radiance Cache 接続

`TraceScreenProbes()` は各レイを以下の順で評価:

1. **Screen Trace**（カメラ空間で SSR 風の depth march）
2. **Mesh SDF Trace**（または Hardware Ray Tracing） → Surface Cache 上で Lighting 評価
3. **Sky / Distant Falloff** または **Radiance Cache 切替**: レイが TMin を越えたら [[lumen_radiance_cache]] をサンプリングして打ち切り

### 3.5 SH3 化と空間フィルタ

`FilterScreenProbes()` 内で:

- 8×8 oct 放射輝度 → **3 次球面調和（SH3）係数 9 個** に投影（`ScreenProbeRadianceSHAmbient`/`SHDirectional`）
- 近傍プローブ間で SH 係数を加重平均（`SpatialFilterProbes=1`）
- ピクセル統合時は SH3 係数からピクセル法線方向の irradiance を解析評価

理由: ノイズ低減と帯域削減（9 ch float vs 64 oct samples）。

### 3.6 時間フィルタ

`FScreenProbeTemporalReprojectionCS`（`LumenScreenProbeGatherTemporal.usf`）:

- 速度バッファで前フレームピクセルへ reprojection
- Variance Clamp で history 範囲外を除外
- `MaxFramesAccumulated=10` で指数移動平均
- **Fast Update Mode**: ライティング変化が激しい領域（`FractionOfLightingMovingForFastUpdateMode=0.1`）は短い history（最大 90% で `MaxFastUpdateModeAmount=0.9`）に切替

---

## 4. 近似差分（厳密 vs UE）

| 項目 | 厳密 PT | Lumen Final Gather |
|------|--------|-------------------|
| Sampling per pixel | ≥ 1024 ray | 64 ray / プローブ × 1/256 ピクセル ≈ 0.25 ray/pixel |
| 動的光源応答 | 即時 | Surface Cache + Radiance Cache に間接、3〜10 frame 遅延 |
| Disocclusion 反応 | 即時 | Adaptive Probe + Fast Update Mode で 1〜3 frame |
| Bias | なし | プローブ補間 → 細部の漏れ・にじみ |
| 光線高周波（Caustics 等） | 表現可 | プローブ低解像度のため失われる（→ HW RT Hit Lighting 必要） |

---

## 5. 主要 CVar / パラメータ

| CVar | 既定 | 効果 |
|------|------|------|
| `r.Lumen.ScreenProbeGather` | 1 | Final Gather 全体の有効化 |
| `r.Lumen.ScreenProbeGather.DownsampleFactor` | 16 | プローブ間隔（pixel）。8 で品質倍・コスト 4 倍 |
| `r.Lumen.ScreenProbeGather.TracingOctahedronResolution` | 8 | プローブあたり ray 数 = N² |
| `r.Lumen.ScreenProbeGather.NumAdaptiveProbes` | 8 | タイルあたり最大追加プローブ数 |
| `r.Lumen.ScreenProbeGather.AdaptiveProbeAllocationFraction` | 0.5 | 一様プローブ数の何倍まで追加を許すか |
| `r.Lumen.ScreenProbeGather.IrradianceFormat` | 1 | 0=SH3 / 1=Octahedral irradiance |
| `r.Lumen.ScreenProbeGather.SpatialFilterProbes` | 1 | プローブ間空間フィルタ |
| `r.Lumen.ScreenProbeGather.Temporal` | 1 | ピクセル時間フィルタ |
| `r.Lumen.ScreenProbeGather.Temporal.MaxFramesAccumulated` | 10 | 履歴最大蓄積フレーム |
| `r.Lumen.ScreenProbeGather.Temporal.MaxFastUpdateModeAmount` | 0.9 | 動的領域での新フレーム重み上限 |
| `r.Lumen.ScreenProbeGather.RadianceCache` | 1 | Radiance Cache でレイ短絡 |
| `r.Lumen.ScreenProbeGather.ShortRangeAO` | 1 | 別途短距離 AO を追加（ScreenProbe 解像度が粗いため） |
| `r.Lumen.ScreenProbeGather.IntegrationTileClassification` | 1 | VGPR 別タイル分類でオキュパンシ向上 |
| `r.Lumen.ScreenProbeGather.ReferenceMode` | 0 | デバッグ: 1024 uniform ray, no filter |

### 品質プリセット

`PostProcessVolume.LumenFinalGatherQuality` でスケール:

- 1.0（既定）: TracingOctRes 8、DS 16
- 6.0+: DS が 16→8 に半減 → 4× コスト、品質倍
- 0.25: TracingOctRes 4、ノイジーだが軽量

---

## 6. 代替手法と比較

| 手法 | 特徴 | 採用例 |
|------|------|--------|
| **Light Probe Volume**（事前ベイク） | 高速・静的のみ | UE4 ILC、UE5 Lumen 無効時 |
| **VXGI**（Voxel Cone Tracing） | 動的だが leaking、VRAM 爆発 | 旧 NVIDIA GameWorks |
| **DDGI**（Dynamic Diffuse GI Probes） | 等間隔プローブ + relocation | RTXGI |
| **ReSTIR GI** | reservoir で per-pixel sampling | [[lumen_restir_gather]] (UE プロト) |
| **Path Tracing Reference** | Ground truth | UE Path Tracer モード |

### Lumen ScreenProbeGather の位置付け

- DDGI が **ワールド空間プローブ** を均等配置するのに対し、Lumen は **画面空間プローブ** を可視ピクセル付近にだけ配置 → カメラから見える範囲に予算集中
- 遠方は [[lumen_radiance_cache]]（ワールド空間）にフォールバックして両者を併用

---

## 7. 参考資料

- **S10**: [Lumen GI Course Notes (SIGGRAPH 2022)](https://advances.realtimerendering.com/s2022/index.html#Lumen)
- Heitz 2018 "A Low-Distortion Map Between Triangle and Square" — Octahedral マッピングの理論背景
- McGuire 2017 "Stochastic Sampling for Real-Time Indirect" — Importance Sampling の系譜
- Sloan 2008 "Stupid Spherical Harmonics Tricks" — SH3 表現の根拠

---

## 8. 相談用フック（未解決事項）

- **DownsampleFactor=16 の根拠**: 葉っぱや髪などのディテールでは Adaptive Probe が大量発生してプローブ予算を圧迫する。S10 が 16 を選んだ実機計測（GTX 1080 等の世代別）の数値 → [[_source_index]] §3 に登録要
- **SH3 vs Octahedral irradiance のトレードオフ**: SH3 は 9ch、Oct は 6×6=36 px。SH3 はリンギング、Oct はメモリ。UE が Oct を既定にした品質基準が不明
- **Adaptive Probe 上限 0.5 倍の根拠**: それ以上は VRAM/coherence 悪化と書かれているが、具体プロファイル数値が S10 にない
