---
name: Lumen ReSTIR Gather
description: ReSTIR GI（Reservoir Spatiotemporal Importance Resampling）の理論と Lumen 試作実装
type: project
---

# Lumen ReSTIR Gather（Reservoir 再サンプリング Final Gather）

- 上位: [[_algorithm_index]]
- 関連: [[lumen_final_gather]] / [[lumen_hw_rt]] / [[lumen_surface_cache]]
- 採用システム: Lumen Diffuse GI（プロトタイプ、`r.Lumen.ReSTIRGather=1` でオプトイン）
- 出典:
  - **S11**: Ouyang et al. 2021 EGSR "ReSTIR GI: Path Resampling for Real-Time Path Tracing"
  - 親: Bitterli et al. 2020 SIGGRAPH "Spatiotemporal reservoir resampling for real-time ray tracing with dynamic direct lighting" (ReSTIR DI 原典)
  - 影響元: Talbot et al. 2005 "Importance Resampling for Global Illumination" (RIS 原典)
  - **S10**: Wright et al. 2022 SIGGRAPH "Lumen" — §11 Future Work で ReSTIR Gather を予告

---

## 1. 何のためのアルゴリズムか

[[lumen_final_gather]] のスクリーンプローブは **8×8=64 サンプル/プローブ** で 1/256 ピクセル分のレイ予算しかなく、**高周波な間接光（小さな強光源、近接の動的光源、Caustics 風）を取り逃がす**。

### 素朴な手法の問題

| 手法 | 問題 |
|------|-----|
| ピクセルあたり 64 ray | 1080p で 130M ray/frame、不可 |
| ピクセルあたり 1 ray + 時間蓄積 | 動的シーンで lag、ノイズ大 |
| 重要サンプリング（BRDF / 環境マップ）のみ | 動的可視性を反映できない |

### ReSTIR GI の貢献（S11）

**Reservoir** という小さな構造に「これまで見たベストサンプル」を確率的に貯め、

- **空間再サンプリング**: 近隣ピクセルの reservoir から自分の視点・法線で評価し直して合流
- **時間再サンプリング**: 前フレームの reservoir を reprojection してマージ

これだけで **ピクセルあたり 1 ray でも数百サンプル相当の有効分散** が得られる。

Lumen ReSTIR Gather は S11 をベースに [[lumen_hw_rt]] を組み合わせた試作版（プラットフォーム D3D12 SM6 のみ）。

---

## 2. 理論

### 2.1 Reservoir 構造（RIS）

各ピクセルは以下を持つ:

```
struct Reservoir {
    Sample sample;   // 採択されたサンプル（ray dir + outgoing radiance + hit dist + hit normal）
    float wSum;      // ストリーム中の重み総和
    float M;         // ストリーム長（観測サンプル数）
    float W;         // RIS 結合重み（最終評価に使う）
}
```

定義: $W = \dfrac{w_\mathrm{sum}}{M \cdot \hat{p}(\mathrm{sample})}$（$\hat{p}$ は target PDF）。

### 2.2 Update（Weighted Reservoir Sampling）

サンプル $x$ を入力ストリームから 1 個だけ重み付き選ぶ:

```
wNew = TargetPDF(x) / OriginalPDF(x)
wSum += wNew
M += 1
if (rand() < wNew / wSum):
    sample = x
```

数学的に: 各サンプル $x_i$ が最終的に reservoir に残る確率は $w_i / \sum_j w_j$ となり、**1 パスで重み付き選択** が可能（Vitter 1985）。

### 2.3 Merge（reservoir 同士の合流）

別の reservoir $R'$ をマージ:

```
wNew' = TargetPDF(R'.sample) × R'.W × R'.M
Update(R, R'.sample, wNew')
M = M_old + R'.M
```

これで 2 つの reservoir が「観測サンプル数 M を維持しつつ」1 つに合流できる。

### 2.4 Spatiotemporal Resampling

| ステージ | 内容 |
|---------|------|
| **Initial Sampling** | RT で 1 ray トレース、reservoir 1 個生成 |
| **Temporal Resampling** | 前フレーム reservoir を speed-buffer で reproject、Merge |
| **Spatial Resampling** | N 回（既定 2）、近傍 K サンプル（既定 4）からランダム選択して Merge |
| **Evaluate** | 最終 reservoir の 1 サンプルを W で評価して放射輝度を出力 |

### 2.5 visibility 検証

reservoir の sample は別ピクセルの ray 由来なので **自分から見ると遮蔽される可能性** がある。Lumen は spatial 再サンプリング時に **screen-space occlusion trace**（`SpatialResamplingOcclusionScreenTraceDistance=0.2` 画面比）で hit position 可視性を検証してから受け入れる（leaking 抑制）。

時間再サンプリングも `ValidateEveryNFrames` で N フレームに 1 回 RT 再トレースして反射輝度の不一致を検出 → 古い reservoir を破棄。

---

## 3. UE 実装

### 3.1 Reservoir テクスチャ群（`LumenReSTIRGather.h:10-24`）

5 枚の R/W テクスチャでひとつの Reservoir を表現:

| テクスチャ | フォーマット | 内容 |
|-----------|------------|------|
| `ReservoirRayDirection` | RGBA16F | xyz=ray dir, w=PDF |
| `ReservoirTraceRadiance` | R11G11B10F | 放射輝度 |
| `ReservoirTraceHitDistance` | R32F | hit 距離 |
| `ReservoirTraceHitNormal` | A2B10G10R10 | hit 法線 |
| `ReservoirWeights` | RGBA16F | x=wSum, y=M, z=W |

### 3.2 パイプライン（`LumenReSTIRGather.cpp:929〜`）

```
RenderLumenReSTIRGather()
├── (1) Allocate Reservoir Textures（DownsampleFactor=2 で半解像度）
├── (2) Validate Temporal Reservoirs（N フレームに 1 回）
│       └── FLumenValidateReservoirsRGS（HW RayGen, LumenReSTIRGather.usf）
├── (3) Initial Sampling
│       └── FLumenInitialSamplingRGS（HW RayGen） → reservoir 生成
├── (4) Temporal Resampling
│       └── FTemporalResamplingCS  → 前フレーム reservoir を reproject + Merge
├── (5) Spatial Resampling × N
│       └── FSpatialResamplingCS（既定 2 パス × 4 samples） → 近傍 reservoir Merge
├── (6) Upsample & Integrate
│       └── FUpsampleAndIntegrateCS（半解像度 reservoir → フル解像度 irradiance）
├── (7) Temporal Accumulation（ピクセル単位、別途）
│       └── FTemporalAccumulationCS
└── (8) Bilateral Filter
        └── FBilateralFilterCS
```

### 3.3 Initial Sampling（HW Ray Tracing 必須）

```hlsl
// LumenReSTIRGather.usf 参照（Initial Sampling 段）
// 1 ピクセルあたり 1 ray、reservoir 初期化
Reservoir.Initialize();
TraceRay(...);  // HW RT
Sample.OutgoingRadiance = ...;  // hit 点の Surface Cache 評価
Reservoir.Update(Sample, TargetPDF / OriginalPDF, Noise);
Reservoir.W = wSum / (M * TargetPDF);
```

`DoesPlatformSupportLumenReSTIRGather` は `SP_PCD3D_SM6` のみ許可（プロトタイプのため他 RHI 未対応）。

### 3.4 Reservoir Merge 実装（`LumenReSTIRGather.usf:187-200`）

```hlsl
bool Merge(FReservoir OtherReservoir, float OtherReservoirSamplePDF, float Noise)
{
    float M0 = M;
    bool bChangedSample = Update(
        OtherReservoir.Sample,
        OtherReservoirSamplePDF * OtherReservoir.W * OtherReservoir.M,
        Noise);
    M = M0 + OtherReservoir.M;
    return bChangedSample;
}
```

`OtherReservoirSamplePDF` は **自分の視点・法線で評価した** target PDF（隣ピクセルの target PDF ではない）→ MIS の正しい再評価。

### 3.5 Spatial Resampling のジオメトリゲート

`LumenReSTIRGather.cpp:115-129` のしきい値で隣 reservoir を棄却:

| パラメータ | 既定 | 役割 |
|-----------|------|------|
| `ResamplingAngleThreshold` | 25° | 法線角差 |
| `ResamplingDepthErrorThreshold` | 1% | 相対深度差 |
| `SpatialResamplingKernelRadius` | 0.05（画面比） | サンプル円半径 |
| `SpatialResamplingOcclusionScreenTraceDistance` | 0.2（画面比） | hit 可視性 trace 長さ |

### 3.6 Upsample + Bilateral Filter

半解像度 reservoir → フル解像度。2 つの方式（`r.Lumen.ReSTIRGather.Upsample.Method`）:

- **0**: jittered bilinear（同一 plane のみ）
- **1**: spiral pattern 16 サンプル（既定）
- **2**: passthrough（debug）

最終 Bilateral Filter（`BilateralFilterCS`）:

- `BilateralFilter.SpatialKernelRadius=0.002`（画面比）
- `BilateralFilter.NumSamples=8`
- 分散が `StrongBlurVarianceThreshold=0.5` を超えるピクセルは強ブラー

---

## 4. 近似差分（厳密 vs UE）

| 項目 | 厳密 PT | ReSTIR GI 論文 (S11) | Lumen ReSTIR Gather |
|------|--------|---------------------|---------------------|
| ray/pixel | 1024+ | 1（+ resample 数百相当） | 0.25（半解像度 1 ray/reservoir） |
| 時間再サンプリング | unbiased | 大数の lag は許容 | + N フレーム再トレース検証 |
| Visibility validate | 完全 | spatial で screen-space occlusion | UE は同方式 + temporal 再トレース |
| Diffuse のみ | n/a | 拡散経路に限定 | 同左、Specular は LumenReflections 別系統 |

### 既知の制限（Lumen 側）

- D3D12 SM6 のみ
- Specular 統合は別パス
- Adaptive Probe / Importance Sampling Rays（[[lumen_final_gather]] の機構）は使用しない
- 品質はまだ ScreenProbeGather に届かないため、既定オフ（CVar 既定 0）

---

## 5. 主要 CVar / パラメータ

| CVar | 既定 | 効果 |
|------|------|------|
| `r.Lumen.ReSTIRGather` | 0 | 機能全体の有効化（プロト） |
| `r.Lumen.ReSTIRGather.DownsampleFactor` | 2 | reservoir テクスチャ解像度（1〜8） |
| `r.Lumen.ReSTIRGather.TemporalResampling` | 1 | 時間再サンプリング |
| `r.Lumen.ReSTIRGather.TemporalResampling.ValidateEveryNFrames` | 0 | N>0 で N フレーム毎に reservoir を再トレース検証 |
| `r.Lumen.ReSTIRGather.SpatialResampling` | 1 | 空間再サンプリング |
| `r.Lumen.ReSTIRGather.SpatialResampling.NumPasses` | 2 | 空間パス回数 |
| `r.Lumen.ReSTIRGather.SpatialResampling.NumSamples` | 4 | 1 パスの近傍サンプル数 |
| `r.Lumen.ReSTIRGather.SpatialResampling.KernelRadius` | 0.05 | 半径（画面比） |
| `r.Lumen.ReSTIRGather.SpatialResampling.OcclusionScreenTraceDistance` | 0.2 | 可視性検証 trace 長 |
| `r.Lumen.ReSTIRGather.ResamplingAngleThreshold` | 25° | 法線角差棄却 |
| `r.Lumen.ReSTIRGather.ResamplingDepthErrorThreshold` | 0.01 | 相対深度差棄却 |
| `r.Lumen.ReSTIRGather.MaxRayIntensity` | 100 | firefly clamp |
| `r.Lumen.ReSTIRGather.Upsample.Method` | 1 | 0=bilinear/1=spiral/2=passthrough |
| `r.Lumen.ReSTIRGather.Upsample.NumSamples` | 16 | spiral 時のサンプル数 |
| `r.Lumen.ReSTIRGather.Upsample.KernelSize` | 3.0 | reservoir texel 単位 |
| `r.Lumen.ReSTIRGather.BilateralFilter` | 1 | 最終 bilateral 有効 |
| `r.Lumen.ReSTIRGather.BilateralFilter.NumSamples` | 8 | bilateral サンプル数 |
| `r.Lumen.ReSTIRGather.Temporal.MaxFramesAccumulated` | 10 | ピクセル時間蓄積上限 |

---

## 6. 代替手法と比較

| 手法 | サンプル/pixel | 動的応答 | 実装複雑度 | UE での位置 |
|------|--------------|---------|----------|------------|
| **ScreenProbeGather** | 0.25（プローブ補間） | 中 | 中 | デフォルト Final Gather |
| **ReSTIR Gather** | 1（reservoir 経由で数百相当） | 高 | 高 | プロト（オプトイン） |
| **DDGI** | 0（プローブ事前計算） | 低（プローブ更新限界） | 中 | 未採用 |
| **Path Tracing** | 1024+ | 即時 | 低 | リファレンス（Path Tracer） |

### ReSTIR Gather の理論的優位

- **Caustics 風の高周波 GI**（小さな強光源、近接光源）が表現可能
- **動的シーン応答** が ScreenProbe より良い（reservoir validate で古い情報を破棄）
- **ピクセル単位の精度** で補間ステライリングなし

### 現状の弱点

- VRAM 帯域: 5 枚 R/W テクスチャ × 全画面
- HW RT 必須（Lumen の SDF Path に対応せず）
- Spatial pass の Kernel Radius が大きいと leaking、小さいとノイズ → ハイパラ調整困難

---

## 7. 参考資料

- **S11**: [Ouyang et al. 2021 "ReSTIR GI"](https://research.nvidia.com/publication/2021-06_restir-gi-path-resampling-real-time-path-tracing)
- Bitterli et al. 2020 "Spatiotemporal reservoir resampling" (ReSTIR DI) — reservoir の原型
- Talbot et al. 2005 "Importance Resampling for Global Illumination" (RIS) — 重み付き再サンプリングの理論
- Wyman & Panteleev 2021 "Rearchitecting Spatiotemporal Resampling for Production" (NVIDIA) — production 化の知見
- **S10**: Lumen SIGGRAPH 2022 — §11 で ReSTIR Gather を Future Work として言及

---

## 8. 相談用フック（未解決事項）

- **target PDF の実装定義**: S11 では BSDF×cos×|Lᵢ| を target にする例が示されているが、UE がどの形を採用しているか `LumenReSTIRGather.usf:55-130` の `TargetPDF` 関数を要精読
- **Spatial pass の MIS 重み**: 単純 Merge は biased になる場合があり、unbiased ReSTIR の MIS（pairwise / generalized resampling）を Lumen が採用しているか未確認
- **DownsampleFactor=2 の根拠**: ScreenProbe の DownsampleFactor=16 との整合をどう取るか。reservoir 全画面と GPU バランスの計測値が S10/S11 にない
- **既定オフの理由**: 品質が ScreenProbeGather に届かない具体ケースの公式評価レポートが見当たらない（社内資料？）
