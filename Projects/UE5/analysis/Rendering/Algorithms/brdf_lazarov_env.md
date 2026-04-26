---
name: Lazarov Env BRDF Approximation (S06)
description: Lazarov 2013 (Call of Duty: Black Ops II) の多項式近似。Pre-Integrated BRDF LUT を多項式とexp近似で置換し、テクスチャアクセスなしで Env BRDF (Scale/Bias) を計算する。Mobile・低スペック向け。
type: project
---

# Lazarov Env BRDF Approximation（多項式 LUT）

- 出典 ID: **S06**（[[_source_index]]）
- UE 実装: `Engine/Shaders/Private/BRDF.ush:630` `EnvBRDFApproxLazarov`
- ステータス: **完了 (2026-04-27)**
- 上位: [[_algorithm_index]] / [[../01_rendering_overview]]

---

## 1. 目的

Karis Split-Sum（[[brdf_split_sum]]）の **Pre-Integrated BRDF LUT** を、テクスチャアクセスなしの **多項式 + 指数関数** で置換する。

採用先:
- Mobile（テクスチャ帯域・ステート切替コストの削減）
- Forward シェーダ（LUT バインドが面倒なパス）
- Editor ビューポートの一部（プレビュー）
- Translucent BasePass（IBL を簡易化したい場合）

GBuffer + LUT が使える Deferred 経路では Karis Split-Sum が標準。Lazarov はその軽量代替。

---

## 2. 理論

### 2.1 LUT を多項式で近似

Karis の Pre-Integrated GF LUT は:

$$
\text{LUT}(N\!\cdot\!V, \alpha) = (\text{Scale}, \text{Bias})
$$

の 2D 関数。Lazarov 2013 は MC で計算した値を **2 つの 4 次元定数ベクトル** $c_0, c_1$ で表される多項式 + exp で近似:

$$
r = \alpha \cdot c_0 + c_1
$$
$$
a_{004} = \min(r_x^2,\,2^{-9.28\,N\!\cdot\!V}) \cdot r_x + r_y
$$
$$
(A, B) = (-1.04,\,1.04) \cdot a_{004} + (r_z,\,r_w)
$$

ここで:
$$c_0 = (-1, -0.0275, -0.572, 0.022)$$
$$c_1 = (1, 0.0425, 1.04, -0.04)$$

これは Lagarde 系の改良版で、UE は「Adaptation to fit our G term」と注記している（コメント `BRDF.ush:633`）。Smith Joint Approx に合わせるためにオリジナルから係数微調整あり。

### 2.2 元の数学的性質

- $\alpha = 0$: $r = c_1 = (1, 0.0425, 1.04, -0.04)$, $a_{004}$ は $\min(1, 2^{-9.28 NoV})$ から導出 → グレージングで急峻な減衰
- $\alpha = 1$: $r = c_0 + c_1 = (0, 0.015, 0.468, -0.018)$ → 全体的にフラットな応答
- $a_{004}$ の `min(r.x², exp2(-9.28*NoV))` が **Fresnel-like** な減衰を再現する核

---

## 3. UE 実装

### 3.1 Lazarov 2D 近似

`BRDF.ush:630-640`:

```hlsl
half2 EnvBRDFApproxLazarov(half Roughness, half NoV)
{
    // [ Lazarov 2013, "Getting More Physical in Call of Duty: Black Ops II" ]
    // Adaptation to fit our G term.
    const half4 c0 = { -1, -0.0275, -0.572, 0.022 };
    const half4 c1 = { 1, 0.0425, 1.04, -0.04 };
    half4 r = Roughness * c0 + c1;
    half a004 = min(r.x * r.x, exp2(-9.28 * NoV)) * r.x + r.y;
    half2 AB = half2(-1.04, 1.04) * a004 + r.zw;
    return AB;
}
```

- 入力: `Roughness, NoV`
- 出力: `(A, B) = (Scale, Bias)`
- コスト: 4 mad + 1 exp2 + 1 min + 1 mad + 1 mad ≈ 8 ALU
- LUT 1 タップと比べて: メモリアクセス0、ALU 増、合計コストはモバイルで有利

### 3.2 EnvBRDFApprox（呼び出し側）

`BRDF.ush:642-651`:

```hlsl
half3 EnvBRDFApprox( half3 SpecularColor, half Roughness, half NoV )
{
    half2 AB = EnvBRDFApproxLazarov(Roughness, NoV);

    // Anything less than 2% is physically impossible and is instead considered to be shadowing
    float F90 = saturate( 50.0 * SpecularColor.g );

    return SpecularColor * AB.x + F90 * AB.y;
}
```

`EnvBRDF`（Karis 版）と同じインターフェースで `Texture2DSampleLevel` を `EnvBRDFApproxLazarov` に置換した形。

### 3.3 派生関数

| 関数 | 用途 |
|------|------|
| `EnvBRDFApproxNonmetal(Roughness, NoV)` | F0=0.04 固定（非金属専用、さらに軽量） |
| `EnvBRDFApproxFullyRough(...)` | Roughness=1 固定（Diffuse += Spec*0.45） |
| `EnvBRDFApprox(F0, F90, Roughness, NoV)` | F90 独立指定（Substrate） |

`EnvBRDFApproxNonmetal` は 2D 多項式の左半分のみ（`r.x`, `r.y` のみ）でさらに ALU を半減。

---

## 4. 近似差分（理想 vs 実装）

| 項目 | Karis Split-Sum LUT | Lazarov 多項式 | 補足 |
|------|--------------------|---------------|------|
| ストレージ | 128×128 R16G16 (~64KB) | 8 定数のみ | メモリ節約 |
| ALU | LUT サンプル | 8 mad/min/exp2 | モバイル有利 |
| 精度 | MC 1024 サンプル相当 | 多項式近似 | 高 Roughness×低 NoV で誤差最大 |
| Smith G 整合 | LUT 内に Smith Joint | UE 用に係数調整 | コメント明記 |
| 多重散乱 | LUT 改造で対応可 | 不対応 | エネルギー欠損あり |
| 異方性 | 等方性のみ | 等方性のみ | 同じ |

精度は Karis 比で誤差 ±2〜5% 程度（Lazarov 元論文の主張）。視覚的には判別困難。

---

## 5. 主要 CVar・選択

| 場所 | 採用関数 |
|------|---------|
| Mobile BasePass | `EnvBRDFApprox` (Lazarov) |
| Forward Renderer | `EnvBRDFApprox` (Lazarov) |
| Translucent BasePass | `EnvBRDFApprox` (Lazarov) |
| Deferred Standard | `EnvBRDF` (Karis LUT) |
| Lumen Reflection | `EnvBRDF` (Karis LUT) |
| `r.ForwardShading` | 1 → Forward 経路で Lazarov |

切替は CVar ではなく **シェーダパスのコンパイル時マクロ**で決定される。

---

## 6. 代替手法

| 手法 | 採用条件 | 出典 / UE 実装 |
|------|---------|--------------|
| **Karis Split-Sum** | LUT 利用可 | [[brdf_split_sum]] / `EnvBRDF` |
| **EnvBRDFApproxNonmetal** | 非金属専用 | `BRDF.ush:659` |
| **EnvBRDFApproxFullyRough** | Roughness=1 固定 | `BRDF.ush:668` |
| **Path Tracer Reference** | GroundTruth 比較 | `PathTracer.usf` |

---

## 7. 参考資料

- **Lazarov 2013** "Getting More Physical in Call of Duty: Black Ops II" SIGGRAPH 2013 Course → `_papers/S06_Lazarov_BlackOps2_2013.pdf`
- **Karis 2013** "Real Shading in Unreal Engine 4" → 比較対象
- **Lagarde, de Rousiers 2014** "Moving Frostbite to PBR" → 派生改良版（Frostbite 2D Approximation）
- 出典 ID **S06** ([[_source_index]] 参照)

---

## 8. 相談用フック（不確かなポイント）

- **係数 $c_0, c_1$ の調整根拠**: コメント「Adaptation to fit our G term」のみ。Lazarov 元論文の係数とは微妙に異なる（UE 系は Smith Joint Approx に合わせて再フィット）。フィッティング MATLAB スクリプトは公開されていない。
- **`exp2(-9.28 * NoV)` の `9.28`**: 経験則。Fresnel グレージング減衰の急峻さを制御するチューニング値。
- **`(-1.04, 1.04)` 定数**: Scale/Bias 範囲を ±1 内に正規化するためのもの。これも経験則。
- **Roughness 0.0 付近の精度**: 鏡面に近い領域では LUT の方が精度が高い。Lazarov は Roughness < 0.1 でわずかに過大評価する傾向（特にグレージング）。
- **`EnvBRDFApproxNonmetal` のさらなる軽量化**: F0=0.04 仮定で半分の係数で済む。ただし金属マテリアルで使うと真っ暗になる。コードパスでの分岐に注意。

---

## 関連ドキュメント

- [[brdf_split_sum]] — Karis Split-Sum LUT（標準）
- [[brdf_fresnel_schlick]] — Fresnel
- [[brdf_smith]] — G 項（係数調整の対象）
- [[../01_rendering_overview]]
