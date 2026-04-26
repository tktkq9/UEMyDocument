---
name: Volumetric Cloud Schneider
description: Schneider 系リアルタイムボリューメトリック雲（Worley/Perlin + raymarch + temporal reconstruct）
type: project
---

# Volumetric Cloud - Schneider 系（リアルタイム体積雲）

- 上位: [[_algorithm_index]]
- 関連: [[atmos_hillaire]] / [[fog_volumetric]]
- 採用システム: VolumetricCloud Component（UE5 既定の体積雲）
- 出典 ID: **S61**（[[_source_index]]）— Schneider 2015 SIGGRAPH "Horizon: Zero Dawn"
- 続編: Schneider 2017 GDC / SIGGRAPH "Nubis: Decima Engine"
- 影響元: Bouthors 2008 "Interactive Multiple Anisotropic Scattering in Clouds"

---

## 1. 何のためのアルゴリズムか

**実雲のような形状を持ち、太陽光の散乱・自己遮蔽・銀縁（silver lining）を含む体積雲** を、毎フレーム動的にレイマーチで描く。

### 素朴な手法の問題

| 手法 | 問題 |
|------|-----|
| 板ポリ（billboard）雲 | 視差なし、自己遮蔽なし、3D 飛行不可 |
| 3D テクスチャ全体（4K³） | VRAM 64 GB、サイズ非現実 |
| 完全 brute-force raymarch（512 step） | 毎ピクセル全画面では 1080p で数百 ms |

### Schneider の貢献

- **Modeling**: 形状を **Worley + Perlin ノイズ** + 高度勾配 + Weather Map（密度・カバレッジ） で合成 → 4K³ 相当を 128³ + 32³ で表現
- **Lighting**: 太陽方向に短距離レイマーチ（self-shadow）、Henyey-Greenstein × 2（前方・後方散乱）、Beer-Powder 関数で銀縁
- **Optimization**: cone-marching、empty-space skipping、低解像度 + Temporal Reconstruction

UE5 は VolumetricCloud Component で **マテリアル経由のカスタム密度関数** + Schneider 系 raymarch を実装。

---

## 2. 理論

### 2.1 雲の体積定義（雲モデリング）

各位置 $\mathbf{p}$ で密度 $\sigma(\mathbf{p})$:

```
σ(p) = Coverage(WeatherMap(p.xy)) 
     × HeightGradient(p.z, CloudType)
     × ShapeNoise(p)        // Perlin-Worley 128³
     × DetailNoise(p)        // Worley 32³ で侵食
```

- **Weather Map**: 2D テクスチャ R=Coverage, G=Precipitation, B=CloudType
- **Height Gradient**: 雲タイプ（cumulus/stratus/cumulonimbus）ごとに高さ方向の密度プロファイル
- **Shape Noise**: 大局形状（128³ 程度の Perlin-Worley）
- **Detail Noise**: 表面侵食用ハイ周波（32³ Worley）

### 2.2 体積積分（Beer's law）

レイ $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ に沿って:

$$
T(t) = \exp\bigg(-\int_0^t \sigma(\mathbf{r}(s)) \, ds\bigg)
$$

$$
L = \int_0^{t_\mathrm{max}} T(t) \cdot \sigma(t) \cdot \big(L_\odot \cdot P(\theta) \cdot T_\odot(t) + L_\mathrm{ambient}\big) \, dt
$$

- $T_\odot(t)$: サンプル点 $t$ から太陽方向への transmittance（**短距離 6 step ライトマーチ** で近似）
- $P(\theta)$: 位相関数

### 2.3 位相関数（2-lobe HG）

```
P(θ) = lerp(HG(θ, g_forward), HG(θ, g_backward), 0.5)
g_forward ≈ +0.8   // 太陽方向
g_backward ≈ -0.5  // 反対方向（やや明るい）
```

Schneider 流の "silver lining"（太陽の縁取り）はこれで再現。

### 2.4 Beer-Powder 関数

太陽方向 transmittance に **Powder 効果**（粉砂糖風の自己照明）を加算:

```
T_eff = exp(-d) − exp(-2d)   // d = optical depth
```

雲端で大きく、内部で 0 に近づく → 入射光側面が明るく見える物理整合な近似。

### 2.5 サンプリング戦略

| 戦略 | 内容 |
|------|------|
| **Cone marching** | 太陽方向は radius を距離とともに広げて低解像度に低下 |
| **Empty-space skipping** | 低解像 conservative-density volume で密度ゼロ領域を一気にスキップ |
| **Adaptive step** | 雲外: 大ステップ / 雲内: 小ステップ。連続 N ゼロサンプルで再加速 |
| **Temporal reproject** | 1/16 ピクセルだけ毎フレーム更新 → 16 frame で 1 完全フレーム再構築 |

---

## 3. UE 実装

### 3.1 主要パス（`VolumetricCloudRendering.cpp`）

```
RenderVolumetricCloud()
├── (1) Empty-Space Skipping volume の準備
│       └── FRenderVolumetricCloudEmptySpaceSkippingCS（VolumetricCloud.usf, EMPTY_SPACE_SKIPPING）
├── (2) Volumetric Render Target の準備（低解像度）
├── (3) Cloud raymarch
│       └── FRenderVolumetricCloudRenderViewCS（compute, default）
│       または FRenderVolumetricCloudRenderViewPs（fallback pixel）
├── (4) Cloud Shadow Map（光源側 raymarch）
│       └── FRenderVolumetricCloudRenderViewPs (SHADOW_DEPTH_SHADER)
├── (5) Cloud Sky AO（半球 AO）
└── (6) 上採点 / Temporal Reconstruct（VolumetricRenderTarget）
```

### 3.2 マテリアルベースの雲モデリング

UE5 では雲の**密度関数自体をマテリアルエディタで定義**:

- マテリアル ShadingModel = `Volume`
- `VolumetricCloud.usf` がマテリアルから `SampleExtinctionCoefficients` / `Albedo` / `EmissiveColor` を呼ぶ
- ノードベースで Perlin-Worley テクスチャ + Weather Map を組み合わせる（`SampleVolumetricCloudShapeAndErosionTextures` 等のヘルパー）

### 3.3 サンプル数（CVar 既定）

| 用途 | CVar | 既定 |
|------|------|------|
| プライマリ視線 | `r.VolumetricCloud.ViewRaySampleMaxCount` | 768 |
| 反射 | `r.VolumetricCloud.ReflectionRaySampleMaxCount` | 80 |
| 太陽方向 light march（プライマリ） | `r.VolumetricCloud.Shadow.ViewRaySampleMaxCount` | 80 |
| 太陽方向 light march（反射） | `r.VolumetricCloud.Shadow.ReflectionRaySampleMaxCount` | 24 |
| Sky AO | `r.VolumetricCloud.SkyAO.TraceSampleCount` | （Component 設定） |
| Cloud Shadow Map | `r.VolumetricCloud.ShadowMap.RaySampleMaxCount` | （Component 設定） |
| Empty-Space Skip | `r.VolumetricCloud.EmptySpaceSkipping` | 1 |
| Empty-Space depth | `r.VolumetricCloud.EmptySpaceSkipping.VolumeDepth` | （CVar） |
| サンプル下限 | `r.VolumetricCloud.SampleMinCount` | 2 |
| 距離→サンプル飽和 | `r.VolumetricCloud.DistanceToSampleMaxCount` | 15 km |

### 3.4 Empty-Space Skipping の仕組み

`FRenderVolumetricCloudEmptySpaceSkippingCS` が低解像度 3D ボリュームを生成し、各セルに **maximum density**（コーナー or center サンプリング）を保存。raymarch 中はまずこのボリュームを参照し、密度 0 セルでは大きく step。

```
If (ConservativeDensity == 0):
    t += LargeStep × StepSizeOnZeroConservativeDensity  // 既定 1
```

### 3.5 Cloud Shadow Map

地表・大気・他オブジェクトに雲の影を落とすため、太陽方向から見た低解像度シャドウマップ生成:

- `r.VolumetricCloud.ShadowMap=1`
- 解像度: `CloudShadowMapResolutionScale × 512`、最大 `r.VolumetricCloud.ShadowMap.MaxResolution`
- 各テクセル: 累積 transmittance + averaged depth
- Spatial filter（`r.VolumetricCloud.ShadowSpatialFiltering`）+ Temporal filter で安定化

### 3.6 Aerial Perspective との合成

雲の各サンプルで [[atmos_hillaire]] の Aerial Perspective Volume をサンプリングして遠方雲を青く減衰。

`r.VolumetricCloud.HighQualityAerialPerspective=0` 既定（AP-LUT 使用）。1 にすると毎サンプル AP raymarch、品質倍 / コスト数倍。

### 3.7 Volumetric Render Target との Temporal Reconstruct

`VolumetricRenderTarget` フレームワーク（`r.VolumetricRenderTarget`）:

- 低解像度（既定 1/4 解像度）でレイマーチ
- 4×4 毎フレームジッタ + 履歴 reproject で full-res 再構築
- Disocclusion 領域は再トレース

これにより 1080p の雲を 270×270 raymarch で済ませる（コスト 1/16）。

### 3.8 Local Lights サンプリング

`r.VolumetricCloud.EnableLocalLightsSampling=1` で、雲内のサンプル点でローカルライト（PointLight 等）も評価。雷光や夜景のためで、コスト増。

---

## 4. 近似差分（厳密 vs UE）

| 項目 | 厳密 PT | Schneider / UE |
|------|--------|---------------|
| 散乱次数 | 多重 | 1 次 + 等方 ambient + Powder 近似 |
| 位相関数 | Mie 完全 | 2-lobe HG |
| サンプル数 | 数千 | 768 + 80 (sun) max |
| 雲モデリング | 物理（droplet 分布） | Perlin-Worley 合成、アーティスト調整 |
| 自己照明（Powder） | 多重散乱から自然発生 | Beer-Powder 近似式 |
| Spatiotemporal | フレーム単位 | 1/16 解像度 + 16 frame reconstruct → 雲が高速移動すると残像 |

---

## 5. 主要 CVar / パラメータ

### Tracing

| CVar | 既定 | 効果 |
|------|------|------|
| `r.VolumetricCloud` | 1 | 全体有効化 |
| `r.VolumetricCloud.ViewRaySampleMaxCount` | 768 | プライマリ最大 step |
| `r.VolumetricCloud.ReflectionRaySampleMaxCount` | 80 | 反射用 |
| `r.VolumetricCloud.SampleMinCount` | 2 | 最少 step |
| `r.VolumetricCloud.SampleClampCount` | INT_MAX | 全体上限（async compute 安定化用） |
| `r.VolumetricCloud.DistanceToSampleMaxCount` | 15 km | この距離で max に達する |
| `r.VolumetricCloud.StepSizeOnZeroConservativeDensity` | 1 | 空セル時の step 倍率 |
| `r.VolumetricCloud.HighQualityAerialPerspective` | 0 | per-sample AP raymarch |

### Shadow

| CVar | 既定 | 効果 |
|------|------|------|
| `r.VolumetricCloud.ShadowMap` | 1 | 雲影マップ |
| `r.VolumetricCloud.ShadowMap.MaxResolution` | 2048 | 最大解像度 |
| `r.VolumetricCloud.ShadowSpatialFiltering` | （0/1） | 空間フィルタ |
| `r.VolumetricCloud.ShadowTemporalFilteringNewFrameWeight` | （float） | 履歴重み |
| `r.VolumetricCloud.Shadow.SampleAtmosphericLightShadowmap` | 1 | 太陽 shadow map と組み合わせ |

### Empty-Space Skipping

| CVar | 既定 | 効果 |
|------|------|------|
| `r.VolumetricCloud.EmptySpaceSkipping` | 1 | 機能有効 |
| `r.VolumetricCloud.EmptySpaceSkipping.VolumeDepth` | （CVar） | 低解像 volume の深度 |
| `r.VolumetricCloud.EmptySpaceSkipping.SampleCorners` | 0/1 | corner sample（保守的） |

### Sky AO

- `r.VolumetricCloud.SkyAO=1`: 空ドームからの遮蔽計算
- `r.VolumetricCloud.SkyAOMaxResolution=512`

### Component プロパティ（`UVolumetricCloudComponent`）

- `LayerBottomAltitude`, `LayerHeight`（典型 5 km / 5 km）
- `TracingStartMaxDistance`, `TracingMaxDistance`（典型 350 km）
- `PlanetRadius=6360 km`
- `SkyLightCloudBottomOcclusion`
- マテリアル参照（雲密度関数）

---

## 6. 代替手法と比較

| 手法 | 特徴 | 採用例 |
|------|------|--------|
| **Skybox Cloud** | 静的 | 旧 AAA |
| **Billboard Cloud** | 軽量、遮蔽なし | モバイル |
| **Volumetric Slice** | 板を 3D 配置 | UE3 旧 |
| **Schneider 2015 (Decima)** | Perlin-Worley + raymarch | Horizon: ZD, UE5, Decima |
| **Path-traced Clouds** | 厳密 | オフライン CG |

### Schneider 系の優位

- **動的太陽位置に対応**: 朝焼け雲・夕焼け雲が同じパイプラインで自然
- **アーティスト制御**: Weather Map で雲分布を絵的に作成可能
- **3D 飛行**: 視差・自己遮蔽が正しいので飛行ゲームに必須

### 弱点

- **高速雲移動 + Temporal Reconstruct** で残像
- **小さな雲ディテール**（cloudlet）は Detail Noise で制限
- **降水・雷光** は別実装が必要（雲そのものは透明体積）

---

## 7. 参考資料

- **S61**: [Schneider & Vos 2015 SIGGRAPH](https://www.guerrilla-games.com/read/the-real-time-volumetric-cloudscapes-of-horizon-zero-dawn)
- Schneider 2017 GDC "Nubis: Authoring Real-Time Volumetric Cloudscapes with the Decima Engine"
- Hillaire 2016 "Physically based atmospheric and volumetric rendering" — UE での先行実装
- Bouthors et al. 2008 "Interactive Multiple Anisotropic Scattering in Clouds" — silver lining 理論
- Wrenninge & Zafar 2011 "Production Volume Rendering"（Pixar）— Beer-Powder 系の系譜

---

## 8. 相談用フック（未解決事項）

- **Beer-Powder 係数の物理対応**: $T = e^{-d} - e^{-2d}$ の係数 2 は経験的か理論的根拠か。Schneider 講演では絵作り側の経験値とされるが Wrenninge の本では多重散乱からの近似として導出可能とある
- **Cone marching の収束**: Light march 6 step で銀縁が再現される条件、もっと厚い雲で誤差増大が出ないか
- **Empty-Space Skipping の SampleCorners**: 1 にすると保守的だがコスト増。0 で center sampling のとき小雲が消える条件
- **VolumetricRenderTarget の reconstruction artifact**: 雲が画面比 60%/sec で移動すると残像、`r.VolumetricRenderTarget.ReprojectionRadiusScale` で抑えられるかケース別測定要
