---
name: Karis Split-Sum IBL Approximation (S01)
description: Karis 2013 の Image-Based Lighting 近似。環境光の鏡面反射積分を「事前畳み込みキューブマップ × 事前計算された G·F LUT」の積に分離して実時間化する。UE の IBL の中核。
type: project
---

# Karis Split-Sum 近似（IBL Specular）

- 出典 ID: **S01**（[[_source_index]]）
- UE 実装: `Engine/Shaders/Private/BRDF.ush:611` `EnvBRDF`, `Engine/Shaders/Private/ReflectionEnvironmentShared.ush`, `ReflectionEnvironmentComputeShaders.usf`
- ステータス: **完了 (2026-04-27)**
- 上位: [[_algorithm_index]] / [[../01_rendering_overview]]

---

## 1. 目的

Image-Based Lighting（IBL）の鏡面反射:

$$
L_\text{spec}(v) = \int_\Omega L_i(l)\,f_r(l,v)\,(\mathbf{n}\cdot\mathbf{l})\,d\omega_l
$$

をリアルタイムで評価可能な形に近似する。完全な MC 積分は不可能なため、**ライト環境** と **BRDF** を分離（"Split Sum"）して各々を事前積分し、実行時はキューブマップ 1 タップ + LUT 1 タップ で済ませる。

採用先:
- Reflection Capture / Sky Light の Specular 寄与
- Lumen Reflections の Final Gather フォールバック
- Substrate Specular の IBL Lobe

---

## 2. 理論

### 2.1 Split-Sum 分離

積分を 2 項に分離:

$$
\int_\Omega L_i\,f_r\,(\mathbf{n}\cdot\mathbf{l})\,d\omega_l \approx
\underbrace{\left( \int_\Omega L_i\,d\omega_l \right)}_{\text{Pre-Filtered Env Map}} \cdot
\underbrace{\left( \int_\Omega f_r\,(\mathbf{n}\cdot\mathbf{l})\,d\omega_l \right)}_{\text{Pre-Integrated BRDF (LUT)}}
$$

### 2.2 第1項: Pre-Filtered Environment Map

**仮定**: 視線 $\mathbf{v} = \mathbf{n}$（観察方向 = 法線方向）。
GGX NDF で重要度サンプリングし、Roughness 別にキューブマップをミップとして畳み込む:

```
Mip[0] : Roughness 0.0   (元のシャープな環境)
Mip[1] : Roughness 0.25
Mip[2] : Roughness 0.50
Mip[3] : Roughness 0.75
Mip[4] : Roughness 1.0   (完全粗い面)
```

実行時は **Reflection Vector R** とラフネスから決まるミップレベルで `SampleLevel(EnvMap, R, mip)` するだけ。

### 2.3 第2項: Pre-Integrated BRDF LUT

法線基準座標で:

$$
\int f_r(l,v)\,(\mathbf{n}\cdot\mathbf{l})\,d\omega_l = F_0\,\text{Scale}(N\!\cdot\!V, \alpha) + F_{90}\,\text{Bias}(N\!\cdot\!V, \alpha)
$$

ここで Scale/Bias は $F_0$ から独立し、$N\!\cdot\!V$ と $\alpha$ のみの 2D 関数。これを **2D テクスチャ（PreIntegratedGF）** として事前計算 → ランタイムで 1 タップで取得。

### 2.4 視線=法線仮定の補正

実視線 $\mathbf{v} \ne \mathbf{n}$ の場合、グレージング角でハイライトの非対称性（伸びた形）が再現できない。Karis はこれを許容誤差とした上で、**Reflection Vector** $R = \text{reflect}(-V, N)$ をサンプル方向に使うことで近似的に修正している。

---

## 3. UE 実装

### 3.1 EnvBRDF（実行時側）

`BRDF.ush:611-620`:

```hlsl
// [Karis 2013, "Real Shading in Unreal Engine 4" slide 11]
half3 EnvBRDF( half3 SpecularColor, half Roughness, half NoV )
{
    // Importance sampled preintegrated G * F
    float2 AB = Texture2DSampleLevel( PreIntegratedGF, PreIntegratedGFSampler, float2( NoV, Roughness ), 0 ).rg;

    // Anything less than 2% is physically impossible and is instead considered to be shadowing 
    float3 GF = SpecularColor * AB.x + saturate( 50.0 * SpecularColor.g ) * AB.y;
    return GF;
}
```

- `AB.x` = Scale 項（F0 にかかる）
- `AB.y` = Bias 項（F90 にかかる）
- `saturate(50 * SpecularColor.g)` で F90 を導出（Karis 標準）

### 3.2 PreIntegratedGF テクスチャ生成

UE では:
- `PreIntegratedSkin.cpp` / `BuildBRDFLut.cpp` 系で初回起動時に GPU で MC を 1024 サンプル走らせて生成
- 解像度 128×128 程度（NoV × Roughness）
- フォーマット PF_G16R16（Scale, Bias の 2 チャネル）

### 3.3 Pre-Filtered Cubemap

`ReflectionEnvironmentComputeShaders.usf` の `FilterPSK` 系シェーダで:
- ミップ毎にラフネス決定（`Mip = Roughness * MaxMip`）
- `ImportanceSampleGGX` で 64〜1024 サンプル
- 重み付き和でキューブマップ各面を畳み込み

### 3.4 EnvBRDF の F0/F90 拡張形

`BRDF.ush:622-628`:

```hlsl
half3 EnvBRDF(half3 F0, half3 F90, half Roughness, half NoV)
{
    float2 AB = Texture2DSampleLevel(PreIntegratedGF, PreIntegratedGFSampler, float2(NoV, Roughness), 0).rg;
    float3 GF = F0 * AB.x + F90 * AB.y;
    return GF;
}
```

Substrate 系で F90 を独立指定する場合のオーバーロード。

---

## 4. 近似差分（理想 vs 実装）

| 項目 | 理想 GroundTruth | UE 実装 | 補足 |
|------|----------------|---------|------|
| 積分分離 | 不可（双線形ではない） | Split-Sum で分離 | 視覚的に許容 |
| 視線=法線仮定 | 全方向対応 | $\mathbf{v}=\mathbf{n}$ で計算 | グレージング誤差 |
| ミップ選択 | 連続 Roughness | 離散ミップ + 線形補間 | バンディング可能性 |
| MC サンプル数 | 数千 | 64〜1024（Pre-Filter）/ 1024（LUT） | TAA で時間平均 |
| 異方性 | 不対応 | 等方性のみ | 異方性は Lumen Reflection で別経路 |
| 多重散乱 | 含む | LUT に含めるか別補正 | UE は Multi-Scattering Compensation あり |

---

## 5. 主要 CVar

| CVar | 既定 | 効果 |
|------|------|------|
| `r.ReflectionEnvironment` | 1 | Reflection Capture の有効化 |
| `r.SkyLight.RealTimeReflectionCapture` | 0 | Sky Light のリアルタイムキャプチャ |
| `r.ReflectionCaptureCompression` | 1 | キューブマップ圧縮（BC6H） |
| `r.ReflectionCaptureResolution` | 128 | キューブマップ解像度（per-face） |
| `r.LightCulling.MaxCapturesPerCluster` | 8 | クラスタあたり最大キャプチャ数 |
| `r.MaterialPreIntegratedBRDF` | 1 | LUT の使用切替（実験用 0=計算） |

---

## 6. 代替手法

| 手法 | 採用条件 | UE 実装 |
|------|---------|---------|
| **Lazarov Env BRDF Approx** | Mobile・LUT なし | [[brdf_lazarov_env]] / `EnvBRDFApprox` |
| **Lumen Reflections** | リアルタイム動的反射 | [[lumen_final_gather]] / Lumen RT |
| **SSR (Screen Space)** | 短距離・スクリーン内反射 | [[ss_ssr]] |
| **Path Tracer Reference** | オフライン GroundTruth | `PathTracer.usf` |

UE では Lumen Reflection が無効/遠距離の場合のフォールバックとして Karis Split-Sum が常に使用される。

---

## 7. 参考資料

- **Karis 2013** "Real Shading in Unreal Engine 4" SIGGRAPH 2013 Course → `_papers/S01_Karis_RealShadingUE4_2013.pdf`
- **Lagarde, de Rousiers 2014** "Moving Frostbite to PBR" → 多重散乱補正の議論
- **Heitz 2014** "Importance Sampling Microfacet-Based BSDFs using the Distribution of Visible Normals" → Pre-Filter の VNDF 改善
- 出典 ID **S01** ([[_source_index]] 参照)

---

## 8. 相談用フック（不確かなポイント）

- **`saturate(50 * SpecularColor.g)` の F90 推定**は Karis 2013 の経験則。「2% 未満を Shadowing と見なす」根拠は SpecularColor が GBuffer に格納される際のレンジ仮定で、PBR ワークフロー全般の慣習値。
- **ミップ数と Roughness 量子化**: Cubemap のミップ数（通常 7〜8）で Roughness レンジを離散化するため、滑らかな Roughness 変化はテクスチャの線形補間に依存する。Roughness 0.1〜0.3 の領域は最高ミップ間の補間で精度が落ちやすい。
- **Multi-Scattering Compensation**: UE 5.x では `EnergyTerms.ush` で多重散乱の補正を別経路で行うが、Split-Sum LUT 自体には含まれていない。Substrate 経路では LUT が拡張される予定（要確認）。
- **異方性 GGX の IBL** には Karis Split-Sum は対応しない（等方性 LUT のみ）。Anisotropy が指定されたマテリアルは Lumen Reflection or 等方性へのフォールバックで処理される。

---

## 関連ドキュメント

- [[brdf_lazarov_env]] — Lazarov 2013 の LUT を使わない多項式近似
- [[brdf_ggx]] — GGX NDF（Pre-Filter で使う）
- [[brdf_smith]] — Smith Joint Approx（LUT 生成時に使う）
- [[brdf_fresnel_schlick]] — Fresnel
- [[lumen_final_gather]] — 動的反射の主経路
- [[../01_rendering_overview]]
