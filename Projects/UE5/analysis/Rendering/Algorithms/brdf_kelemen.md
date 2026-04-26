---
name: Kelemen Visibility (S07)
description: Kelemen 2001 の "couple specular-matte BRDF" の幾何項近似。VoH のみに依存する超軽量 Visibility 項で、皮膚（Pre-Integrated Skin）や特定のレガシーパスで使用される。
type: project
---

# Kelemen Visibility（VoH ベース幾何項）

- 出典 ID: **S07**（[[_source_index]]）
- UE 実装: `Engine/Shaders/Private/BRDF.ush:366` `Vis_Kelemen`
- ステータス: **完了 (2026-04-26)**
- 上位: [[_algorithm_index]] / [[../01_rendering_overview]]

---

## 1. 目的

幾何項を **VoH のみ** に依存する形で近似することで:

- ALU 最小（1 mul + 1 rcp）
- $N\!\cdot\!L$, $N\!\cdot\!V$ に依存しない安定性（グレージングでも値が破綻しにくい）
- 皮膚特化シェーディング（Pre-Integrated Skin）で **Specular の単純化** に使用

UE では限定用途（Subsurface Profile レガシー、Eye、ClearCoat の一部レガシー経路）。

---

## 2. 理論

### 2.1 元の式（Kelemen 2001）

Kelemen-Szirmay-Kalos 2001 の論文で提案された coupled specular-matte モデルの幾何項:

$$
V_\text{Kelemen}(\mathbf{v}, \mathbf{h}) = \frac{1}{4\,(\mathbf{v}\cdot\mathbf{h})^2}
$$

これは Smith G2 / Cook-Torrance 系の幾何項とは導出が異なるが、**経験的に皮膚の近似で実用的**だった。物理的には正しくないが、コストの低さとアーティスト調整しやすさが利点。

### 2.2 NaN 防止

$\mathbf{v}\cdot\mathbf{h} = 0$ で発散するため、UE は分母にバイアスを加える:

$$
V_\text{Kelemen}^{UE} = \frac{1}{4\,(\mathbf{v}\cdot\mathbf{h})^2 + 10^{-5}}
$$

---

## 3. UE 実装

`BRDF.ush:365-370`:

```hlsl
// [Kelemen 2001, "A microfacet based coupled specular-matte brdf model with importance sampling"]
float Vis_Kelemen( float VoH )
{
    // constant to prevent NaN
    return rcp( 4 * VoH * VoH + 1e-5);
}
```

- 入力: $V\!\cdot\!H$
- 出力: $V$ 項（D · F とで掛け合わせる）
- コスト: 2 mul + 1 mad + 1 rcp ≈ 4 ALU（Smith Joint Approx の半分以下）

### 3.1 採用パス

UE で `Vis_Kelemen` が呼ばれる箇所:

| シェーディングモデル | ファイル | 理由 |
|--------------------|---------|------|
| **Pre-Integrated Skin** | `ShadingModelsMaterial.ush`（旧パス） | 皮膚 Specular の軽量化 |
| **Eye**（一部経路） | `EyeShading.ush` | 角膜 Specular の単純化 |
| **Hair**（古い実装の Specular Component） | `HairBsdf.ush`（旧） | Highlights 計算 |
| **Subsurface（Two-Sided Foliage 等）** | レガシー経路 | 葉の半透過 Specular |

UE 5.x では **Substrate 化**でほとんどが `Vis_SmithJointApprox` に置換されたが、互換維持のため残存。

---

## 4. 近似差分（理想 vs 実装）

| 項目 | 理想 Smith G2 | Kelemen | 補足 |
|------|---------------|---------|------|
| 入力 | $\alpha, N\!\cdot\!V, N\!\cdot\!L$ | $V\!\cdot\!H$ のみ | グレージング安定 |
| Roughness 依存性 | あり | **なし** | Roughness を通す経路で再現性低 |
| 物理的整合性 | Smith は厳密 | 経験則 | 皮膚で許容 |
| VoH=0 | 有限 | 1e-5 でクランプ | 数値安定 |
| 適合 BRDF | Microfacet 全般 | 皮膚・ClearCoat 補助 | 限定用途 |

Roughness が変化しても V 項が変わらないため、ハイライトの「広がり」は D 項（GGX）のみで決まる。

---

## 5. 主要 CVar

`Vis_Kelemen` 自体に CVar はない。シェーディングモデル切替で間接的に有効化される:

| シェーディングモデル | マテリアル設定 | Visibility |
|--------------------|--------------|-----------|
| `Default Lit` | 標準 | `Vis_SmithJointApprox` |
| `Preintegrated Skin` | 旧 | `Vis_Kelemen`（限定） |
| `Eye` | 角膜 | `Vis_Kelemen`（限定） |
| `Subsurface Profile` | 主流 | `Vis_SmithJointApprox`（Burley 系） |

`r.PostProcessing.PreserveAlphaChannel` 等の CVar は無関係。

---

## 6. 代替手法

| 手法 | 関数 | 採用条件 |
|------|------|---------|
| **Smith Joint Approx** | `Vis_SmithJointApprox` | 標準（[[brdf_smith]]） |
| **Schlick Visibility** | `Vis_Schlick` | レガシーモバイル（[[brdf_schlick_vis]]） |
| **Smith Joint Full** | `Vis_SmithJoint` | Path Tracer 参照 |
| **Implicit V = 0.25** | `Vis_Implicit` | 平坦化（実験用） |

---

## 7. 参考資料

- **Kelemen, Szirmay-Kalos 2001** "A Microfacet Based Coupled Specular-Matte BRDF Model with Importance Sampling" Eurographics 2001 → `_papers/S07_Kelemen_2001.pdf`
- **Penner 2011** "Pre-Integrated Skin Shading" GPU Pro 2 → 皮膚での Kelemen 採用文脈
- 出典 ID **S07** ([[_source_index]] 参照)

---

## 8. 相談用フック（不確かなポイント）

- **物理的根拠の薄さ**: Kelemen は元論文で「coupled specular-matte」モデルの幾何項として導出。Smith や Schlick とは導出経路が異なり、Microfacet 厳密理論からの帰結ではない。実験的には皮膚で許容、金属では誤差大。
- **UE 5.x での採用範囲**: Substrate 化の進展で、純粋な `Vis_Kelemen` 経路はほぼレガシー（Pre-Integrated Skin の旧モデル）。新規プロジェクトでは Subsurface Profile（Burley + Smith）を推奨。
- **Roughness 依存性の欠如**: D 項（GGX）が Roughness を制御する一方、V 項は VoH のみ。広範な Roughness レンジで挙動が一貫しないため、皮膚以外の素材で使うとアーティファクト（鏡面ハイライトの幅と暗さの整合不良）。
- **`1e-5` クランプの根拠**: VoH=0 で発散するため安全マージンとしての小定数。理論値ではない。

---

## 関連ドキュメント

- [[brdf_smith]] — Smith Joint Approx（標準幾何項）
- [[brdf_schlick_vis]] — Schlick Visibility（軽量版）
- [[sss_burley]] / [[sss_separable]] — 皮膚拡散散乱（V 項とは独立）
- [[../01_rendering_overview]]
