---
name: SkyAtmosphere Hillaire 2020
description: Hillaire 2020 物理大気散乱モデル（4 LUT 構成）の理論と UE 実装
type: project
---

# SkyAtmosphere - Hillaire 2020（物理大気散乱）

- 上位: [[_algorithm_index]]
- 関連: [[atmos_clouds]] / [[fog_volumetric]]
- 採用システム: SkyAtmosphere Component（UE5 既定の空・大気）
- 出典:
  - **S60**: Hillaire 2020 EGSR "A Scalable and Production Ready Sky and Atmosphere Rendering Technique"
  - 親: Bruneton & Neyret 2008 EG "Precomputed Atmospheric Scattering"（事前計算 4D LUT）
  - 影響元: Nishita et al. 1993「最初の物理大気モデル」

---

## 1. 何のためのアルゴリズムか

**地球大気を光線が通過する際の散乱・吸収** を数値積分し、空の色・地平線のぼかし・地上から見た太陽光の減衰（atmospheric extinction）・遠方物体の青みがかり（aerial perspective）を物理ベースで描画する。

### 素朴な手法の問題

| 手法 | 問題 |
|------|-----|
| キューブマップ事前ベイク | 太陽位置・時刻変動に対応不可 |
| Bruneton 2008 4D LUT | 事前計算重い、起動時に数百 MB |
| 完全ブルートフォース raymarch | 1 ピクセルあたり数百 step、4K リアルタイム不可 |

### Hillaire 2020 の貢献（S60）

**4 つの軽量 LUT** に分解して動的に毎フレーム再計算（事前ベイク不要・太陽位置を自由に動かせる）:

| LUT | 解像度 | 役割 |
|-----|------|------|
| Transmittance LUT | 256×64 | 大気を通る光の減衰率 |
| Multi-Scattering LUT | 32×32 | 多重散乱の近似（Hillaire の Isotropic 近似） |
| Sky-View LUT | 192×104 | カメラから見た空ドームの放射輝度 |
| Aerial Perspective Volume | 32×32×16 | カメラ前方フラスタムの 3D ボリューム（フォグとシーン物体の合成） |

→ ピクセル単位ライティングは `LUT 4 回 lookup` だけで完結、4K@60fps 可能。

---

## 2. 理論

### 2.1 大気の散乱モデル

各高度 $h$ で:

| 項目 | 関数 | 物理 |
|------|------|------|
| **Rayleigh 散乱** | $\beta_R(h) = \beta_R^0 \exp(-h/H_R)$ | 短波長（青）優位、空気分子。$H_R \approx 8 km$ |
| **Mie 散乱** | $\beta_M(h) = \beta_M^0 \exp(-h/H_M)$ | エアロゾル、太陽近傍に明るい haze。$H_M \approx 1.2 km$ |
| **Mie 吸収** | $\beta^a_M(h)$ | エアロゾル吸収 |
| **オゾン吸収** | tent function $20-40km$ | 紫色・赤の夕焼けに必要 |

合計 extinction: $\beta_t(h) = \beta_R(h) + (\beta_M(h) + \beta^a_M(h)) + \beta^a_O(h)$

### 2.2 位相関数

| 種類 | 式 |
|------|---|
| **Rayleigh phase** | $P_R(\mu) = \frac{3}{16\pi}(1 + \mu^2)$ |
| **Mie phase**（Henyey-Greenstein, $g \approx 0.8$） | $P_M(\mu) = \frac{1-g^2}{4\pi(1+g^2-2g\mu)^{3/2}}$ |

$\mu = \cos\theta$（光線と入射光のなす角の cos）。

### 2.3 単散乱積分

カメラから方向 $\omega$ にレイマーチして:

$$
L(\omega) = \int_0^{t_\mathrm{max}} T(0 \to t) \cdot S(t, \omega) \, dt
$$

- $T(0 \to t) = \exp\big(-\int_0^t \beta_t \, ds\big)$: **Transmittance**（光の通過率）
- $S(t, \omega) = \beta(t) \cdot P(\theta) \cdot L_\odot \cdot T(\mathrm{sun} \to t)$: 散乱項（太陽光が高さ $h(t)$ まで届く Transmittance × Rayleigh+Mie 位相 × extinction）

**核心**: 太陽方向の Transmittance を毎 step 計算するのが重いので、これを **2D LUT (高度 × 太陽天頂角)** にして lookup する（= Transmittance LUT）。

### 2.4 多重散乱の近似

二次以上の散乱は本来 5D 積分。Hillaire は **等方近似**:

> 「ある点における多重散乱寄与は、あらゆる方向から等方的に届く」

として 2D LUT (高度 × 太陽天頂角) で近似（= Multi-Scattering LUT）。1 ピクセル分の積分球は 64 サンプル（HighQuality 時）または 2 サンプル（既定）。

### 2.5 Sky-View LUT

カメラから見た空ドームを **緯度経度** で 192×104 にサンプリング。各テクセル = 「その方向の空の最終放射輝度」。空の描画は単に 1 lookup。

LUT は **非線形パラメタライズ**（地平線付近に解像度集中）で 192×104 でも十分滑らか。

### 2.6 Aerial Perspective Volume

カメラフラスタムを 3D テクスチャ（典型 32×32×16）に分割。各 froxel:

- カメラからの距離（対数分布、近 4 m〜遠 96 km）
- 内容: 累積散乱光 $L_\mathrm{scattered}$ と Transmittance $T$（RGBA、最大 96 km まで）

シーンピクセル合成時は深度から 3D UVW を作り 1 lookup → `SceneColor * T + L_scattered` でフォグ合成。

---

## 3. UE 実装

### 3.1 全体パス（`SkyAtmosphereRendering.cpp`）

```
RenderSkyAtmosphereLookUpTables()
├── (1) RenderTransmittanceLutCS              ← FRenderTransmittanceLutCS
├── (2) RenderMultiScatteredLuminanceLutCS    ← FRenderMultiScatteredLuminanceLutCS
├── (3) RenderDistantSkyLightLutCS            ← FRenderDistantSkyLightLutCS（環境光）
├── (4) RenderSkyViewLutCS                    ← FRenderSkyViewLutCS
└── (5) RenderCameraAerialPerspectiveVolumeCS ← FRenderCameraAerialPerspectiveVolumeCS

RenderSkyAtmosphere()
└── FRenderSkyAtmospherePS / RenderSkyAtmosphereRayMarchingPS
    ↑ 上空ピクセル = SkyViewLut lookup、地形近接 = raymarch fallback
```

各 CS は `SkyAtmosphere.usf` に実装（`SkyAtmosphereCommon.ush` で散乱関数を共有）。

### 3.2 LUT 解像度（CVar 既定）

| LUT | 既定 | CVar |
|-----|------|------|
| Transmittance | 256×64 | `r.SkyAtmosphere.TransmittanceLUT.{Width,Height}` |
| Multi-Scatter | 32×32 | `r.SkyAtmosphere.MultiScatteringLUT.{Width,Height}` |
| Sky-View | 192×104 | `r.SkyAtmosphere.FastSkyLUT.{Width,Height}` |
| Aerial Perspective | 32×32×16 | `r.SkyAtmosphere.AerialPerspectiveLUT.Width=32`, `DepthResolution=16` |

### 3.3 大気パラメータ（`USkyAtmosphereComponent`）

`FAtmosphereUniformShaderParameters` に bind:

```hlsl
// SkyAtmosphereCommon.ush:312-334
const float DensityMie = exp(Atmosphere.MieDensityExpScale * SampleHeight);
const float DensityRay = exp(Atmosphere.RayleighDensityExpScale * SampleHeight);
s.ScatteringMie = DensityMie * Atmosphere.MieScattering.rgb;
s.AbsorptionMie = DensityMie * Atmosphere.MieAbsorption.rgb;
s.ScatteringRay = DensityRay * Atmosphere.RayleighScattering.rgb;
s.Scattering = s.ScatteringMie + s.ScatteringRay + s.ScatteringOzo;
```

### 3.4 Transmittance LUT 生成（疑似コード）

```hlsl
// FRenderTransmittanceLutCS のロジック
[numthreads(8,8,1)]
void RenderTransmittanceLutCS(uint2 tid)
{
    float ViewHeight, CosViewZenith;
    UvToTransmittanceLutParams(uv, ViewHeight, CosViewZenith);  // 非線形 UV → 物理量

    float3 P = float3(0, 0, ViewHeight);
    float3 dir = float3(sin(zen), 0, cos(zen));
    float3 transmittance = 1;
    for (int i = 0; i < SAMPLE_COUNT; i++)  // 既定 10 サンプル
    {
        float3 sample = ray_step(P, dir, i);
        transmittance *= exp(-extinction(sample) * step_length);
    }
    OutputUAV[tid] = transmittance;
}
```

`r.SkyAtmosphere.TransmittanceLUT.SampleCount=10`。地球規模なので 10 step でも収束する（指数密度のため）。

### 3.5 Sky-View LUT 生成

各 192×104 テクセル = ある (緯度, 経度) 方向の空。レイマーチで:

- 単散乱を MS-LUT で補強しつつ加算
- レイが地面に当たれば打ち切り（black ground または地表 albedo）
- サンプル数は距離適応: `FastSkyLUT.SampleCountMin=4, Max=32`、`DistanceToSampleCountMax=150 km`

### 3.6 Aerial Perspective Volume 生成

各 32×32×16 froxel:

- フラスタム空間でカメラから対数分布距離
- 各 froxel に到達するまでの累積散乱光 + 累積 Transmittance を蓄積
- `SampleCountMaxPerSlice=2` × `DepthResolution=16` = 最大 32 step/pixel

シーン描画時は `SkyAtmosphere.usf` の `RenderSkyAtmosphereRayMarchingPS` で:

```hlsl
float3 UVW = ComputeAerialPerspectiveVolumeUVW(ScreenPos, SceneDepth);
float4 AP = AerialPerspectiveVolume.SampleLevel(UVW, 0);
SceneColor.rgb = SceneColor.rgb * AP.a + AP.rgb;  // T + L
```

### 3.7 Sky の最終描画

ピクセル単位:

1. **Sky pixel**（深度 = far plane、空ドーム）→ Sky-View LUT lookup
2. **遠方シーン** → Aerial Perspective Volume lookup
3. **近接 / 高品質** → raymarch（`RenderSkyAtmosphereRayMarchingPS`、最大 `SampleCountMax=32`）

### 3.8 Distant Sky Light LUT

スカイライト環境光成分用（`DistantSkyLightLUT`、典型高度 6 km から積分）。SH3 / 立方体マップに焼いて全シーンの間接光に使用。`r.SkyAtmosphere.DistantSkyLightLUT.Altitude=6.0`（巻雲高度想定）。

---

## 4. 近似差分（厳密 vs UE）

| 項目 | 厳密 | Hillaire 2020 / UE |
|------|------|-------------------|
| 散乱次数 | 無限次 | 1 次 + 等方多重散乱 LUT |
| 大気プロファイル | 任意 | Earth standard atmosphere パラメタライズ |
| 位相関数 | 正確な Mie | Rayleigh closed form + Mie HG 近似 |
| 太陽 disk 形状 | 角直径 0.53° 円 | 同左、limb darkening は近似 |
| 雲との結合 | 完全結合 | 別パス（[[atmos_clouds]]）で合成 |
| 月光・星 | あり | 別途実装（このパスでは扱わない） |

### Bruneton 2008 との差

- **動的太陽位置**: Bruneton は 4D LUT 起動時ベイク → 太陽方向動かすと再ベイク数秒。Hillaire は毎フレーム再生成 < 0.5 ms
- **VRAM**: Bruneton 数百 MB → Hillaire 数 MB
- **品質**: 多重散乱が Bruneton の正確な再帰積分に対し Hillaire は等方近似 → 朝夕の地平線色味で微差

---

## 5. 主要 CVar / パラメータ

### LUT 解像度・品質

| CVar | 既定 | 効果 |
|------|------|------|
| `r.SkyAtmosphere.TransmittanceLUT` | 1 | Transmittance LUT 生成有効 |
| `r.SkyAtmosphere.TransmittanceLUT.{Width,Height}` | 256, 64 | 解像度 |
| `r.SkyAtmosphere.TransmittanceLUT.SampleCount` | 10 | レイマーチ step |
| `r.SkyAtmosphere.MultiScatteringLUT.{Width,Height}` | 32, 32 | 解像度 |
| `r.SkyAtmosphere.MultiScatteringLUT.SampleCount` | 15 | 多重散乱方向サンプル |
| `r.SkyAtmosphere.MultiScatteringLUT.HighQuality` | 0 | 1 で 64 サンプル |
| `r.SkyAtmosphere.FastSkyLUT.{Width,Height}` | 192, 104 | 解像度 |
| `r.SkyAtmosphere.FastSkyLUT.{SampleCountMin,Max}` | 4, 32 | 距離適応 step |
| `r.SkyAtmosphere.FastSkyLUT.DistanceToSampleCountMax` | 150 (km) | 最大 step に達する距離 |
| `r.SkyAtmosphere.AerialPerspectiveLUT.Width` | 32 | 横解像度（深度方向は別 CVar） |
| `r.SkyAtmosphere.AerialPerspectiveLUT.DepthResolution` | 16 | 深度スライス数 |
| `r.SkyAtmosphere.AerialPerspectiveLUT.Depth` | 96 (km) | 最遠スライス距離 |
| `r.SkyAtmosphere.AerialPerspectiveLUT.SampleCountMaxPerSlice` | 2 | スライスごとの step 上限 |
| `r.SkyAtmosphere.AerialPerspectiveLUT.FastApplyOnOpaque` | 1 | 不透明にも AP-LUT を使用（高速） |
| `r.SkyAtmosphere.LUT32` | 0 | 1 で全 LUT を 32bit/ch に昇格 |
| `r.SkyAtmosphereAsyncCompute` | 0 | LUT 計算を async compute に |

### Component プロパティ（`USkyAtmosphereComponent`）

- `RayleighScattering` (RGB), `RayleighScatteringScale`, `RayleighExponentialDistribution=8 km`
- `MieScattering`, `MieAbsorption`, `MieAnisotropy=0.8`, `MieExponentialDistribution=1.2 km`
- `OzoneAbsorption`, `BottomRadius=6360 km`, `AtmosphereHeight=100 km`
- `GroundAlbedo`, `MultiScatteringFactor`

---

## 6. 代替手法と比較

| 手法 | 特徴 | 採用例 |
|------|------|--------|
| **Static Cubemap** | 起動時静的 | UE4 旧 SkyDome |
| **Preetham 1999** | 解析式、太陽 + 空 | 多くのリアルタイム旧世代 |
| **Hosek-Wilkie 2012** | Preetham 改良 | Blender Cycles 等 |
| **Bruneton 2008** | 4D LUT 高品質 | 古典 AAA、研究 |
| **Hillaire 2020** | 4 LUT 軽量、動的 | UE5 / Frostbite / Decima |
| **Path Traced Atmosphere** | 厳密 | Octane 等オフライン |

### Hillaire 2020 の実装的優位

- **動的太陽位置（Time of Day）** 1 秒で 24h 変化 OK
- **コンポーネント API**: アーティストが Rayleigh/Mie 等を直接調整
- **Volumetric Cloud / Volumetric Fog との統合**: Aerial Perspective Volume を共通参照
- **Ground Reflection**: 地表 albedo からの間接光を MS-LUT で取り込む

---

## 7. 参考資料

- **S60**: [Hillaire 2020 EGSR PDF](https://sebh.github.io/publications/egsr2020.pdf)
- Hillaire のサンプル実装（GitHub）: <https://github.com/sebh/UnrealEngineSkyAtmosphere>
- Bruneton & Neyret 2008 "Precomputed Atmospheric Scattering"（事前計算 LUT 系の親）
- Nishita et al. 1993 "Display of the Earth Taking into Account Atmospheric Scattering"
- Schüler 2012 "An Approximation to the Chapman Grazing-Incidence Function for Atmospheric Scattering"

---

## 8. 相談用フック（未解決事項）

- **多重散乱の等方近似誤差**: 朝夕で実測写真と LUT 結果の差異。S60 §5.2 で議論あるが UE 既定 SampleCount=15 での誤差量を測定したい
- **Sky-View LUT の非線形パラメタライズ**: 地平線集中の式（acos 系？）、`SkyAtmosphereCommon.ush` の `UvToSkyViewLutParams` の正確な式と Hillaire 論文 §5.3 との一致確認
- **Aerial Perspective の near plane 4 m**: なぜ近接 4 m で十分か、近距離 fog がボリューメトリックフォグ（[[fog_volumetric]]）と二重カウントしないか実装上の境界
- **r.SkyAtmosphere.LUT32 を有効化する基準**: HDR スコープでバンディングが見える条件（高ダイナミックレンジ太陽？）の具体ケース
