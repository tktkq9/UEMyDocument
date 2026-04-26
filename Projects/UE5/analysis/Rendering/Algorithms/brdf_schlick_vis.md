---
name: Schlick Visibility (S05)
description: Schlick 1994 の幾何項近似。Smith G2 をパラメータ k = α/2 で線形に近似することで、Smith Joint Approx よりさらに軽量な Visibility 項を提供。低スペック・モバイル向けの代替実装。
type: project
---

# Schlick Visibility（幾何項の Schlick 近似）

- 出典 ID: **S05**（[[_source_index]]）
- UE 実装: `Engine/Shaders/Private/BRDF.ush:374` `Vis_Schlick`
- ステータス: **完了 (2026-04-26)**
- 上位: [[_algorithm_index]] / [[../01_rendering_overview]]

---

## 1. 目的

GGX BRDF の幾何項 $G_2 / (4\,N\!\cdot\!L\,N\!\cdot\!V)$ を **Schlick 1994** の Beckmann 近似経由で軽量化する。Smith G2 の sqrt が削れて mul/mad のみで完結する。

採用先:
- Mobile・低スペックパス
- Forward Renderer の一部（旧パス）
- アーティスト試行用の代替

通常パスでは [[brdf_smith]] (Smith Joint Approx) が標準。

---

## 2. 理論

### 2.1 元の Smith G2 形

$$
G_2(\mathbf{l},\mathbf{v}) = \frac{2}{1 + \sqrt{1 + \alpha^2 \tan^2\theta_v}} \cdot \frac{2}{1 + \sqrt{1 + \alpha^2 \tan^2\theta_l}}
$$

これに対し Schlick の Beckmann 近似式（Schlick 1994）は:

$$
G_1(\mathbf{n}, \mathbf{v}) = \frac{N\!\cdot\!V}{N\!\cdot\!V (1 - k) + k}, \quad k = \alpha\,\sqrt{\frac{2}{\pi}}
$$

UE は Smith GGX の挙動に合わせて **`k = α/2`** に調整（`BRDF.ush:373` のコメント "Tuned to match behavior of Vis_Smith"）。

### 2.2 Visibility 化（除算込み）

BRDF の specular は $D \cdot G_2 \cdot F / (4\,N\!\cdot\!L\,N\!\cdot\!V)$ なので、Visibility = $G_2 / (4\,N\!\cdot\!L\,N\!\cdot\!V)$ を一気に評価する形に変形:

$$
V_\text{Schlick} = \frac{0.25}{\big( N\!\cdot\!V\,(1-k) + k \big)\big( N\!\cdot\!L\,(1-k) + k \big)}
$$

ここで $k = \sqrt{\alpha^2}/2 = \alpha/2$。

---

## 3. UE 実装

`BRDF.ush:373-380`:

```hlsl
// Tuned to match behavior of Vis_Smith
// [Schlick 1994, "An Inexpensive BRDF Model for Physically-Based Rendering"]
float Vis_Schlick( float a2, float NoV, float NoL )
{
    float k = sqrt(a2) * 0.5;
    float Vis_SchlickV = NoV * (1 - k) + k;
    float Vis_SchlickL = NoL * (1 - k) + k;
    return 0.25 / ( Vis_SchlickV * Vis_SchlickL );
}
```

- 入力: $\alpha^2$, $N\!\cdot\!V$, $N\!\cdot\!L$
- 出力: $V = G_2/(4\,N\!\cdot\!L\,N\!\cdot\!V)$
- コスト: 1 sqrt + 4 mul + 2 mad + 1 rcp ≈ 8 ALU
- 比較対象 [[brdf_smith]] (`Vis_SmithJointApprox`): 1 sqrt + 6 mul + 1 rcp ≈ 9 ALU（ほぼ同等）

UE 5.x ではコスト差が消えたため Smith Joint Approx が標準。Schlick はレガシーモバイル経路と互換目的で残存。

---

## 4. 近似差分（理想 vs 実装）

| 項目 | 理想 Smith G2 | Schlick 近似 | 補足 |
|------|---------------|------------|------|
| 形式 | sqrt 2 つを含む有理式 | 線形項 + 線形項の積で除算 | 計算負荷低 |
| $\alpha=0$ | $G_2 = 1$ | $V = 0.25/(N\!\cdot\!V\,N\!\cdot\!L)$ | 一致 |
| $\alpha=1$ | $G_2 \approx 0$ at grazing | 同様にグレージングで小 | 視覚一致 |
| $k = α\sqrt{2/\pi}$ vs `α/2` | Schlick 元 | UE 調整版 | Smith に合わせるためのチューニング |
| エネルギー保存 | Smith は厳密 | Schlick は近似誤差あり | グレージングで弱体化 |

---

## 5. 主要 CVar・選択

UE 5.x ではグローバル切替の CVar はなく、コードパスベースで使用:

| 場所 | 使用関数 |
|------|---------|
| Deferred Standard | `Vis_SmithJointApprox` |
| Forward / Mobile (一部) | `Vis_Schlick` |
| Path Tracer | `Vis_SmithJoint`（フル評価） |
| Hair / Anisotropy | `Vis_SmithJointAniso` |

切替は `BRDF.ush` の `#if SHADING_PATH_*` 系のマクロで行われる。

---

## 6. 代替手法

| 手法 | 関数 | 採用条件 |
|------|------|---------|
| **Smith Joint Approx (Heitz 2014)** | `Vis_SmithJointApprox` | 標準 Deferred ([[brdf_smith]]) |
| **Smith Joint (Full)** | `Vis_SmithJoint` | Path Tracer・GroundTruth 比較 |
| **Smith Joint Anisotropic** | `Vis_SmithJointAniso` | Hair・Anisotropy ([[brdf_ggx_aniso]]) |
| **Implicit V**（古典） | `Vis_Implicit` = 0.25 | F のみで近似する用途 |
| **Neumann V** | `Vis_Neumann` | 金属的応答を強調する非物理的代替 |
| **Kelemen V** | `Vis_Kelemen` | 拡散専用ハイライト（[[brdf_kelemen]]） |

---

## 7. 参考資料

- **Schlick 1994** "An Inexpensive BRDF Model for Physically-Based Rendering" Eurographics → `_papers/S05_Schlick_1994.pdf`
- **Heitz 2014** "Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs" JCGT → 比較
- **Karis 2013** "Real Shading in Unreal Engine 4" SIGGRAPH 2013 → UE が Schlick から Smith に移行した経緯
- 出典 ID **S05** ([[_source_index]] 参照)

---

## 8. 相談用フック（不確かなポイント）

- **`k = α/2` の根拠**: コードコメントは「Tuned to match behavior of Vis_Smith」とのみ。Karis 2013 の経験則で、α=0.4..0.6 範囲で Smith とほぼ一致する。Schlick オリジナル `k = α√(2/π) ≈ 0.798α` とは異なる。
- **モバイル経路の Vis 選択**は実機・ビルドで揺れる。`MobileBasePassPixelShader.usf` の経路を見ないと確定しない（要確認）。
- **Smith vs Schlick で見た目差**: 中域 Roughness では視覚差ほぼ無し。Roughness ≈ 1.0 のグレージングで Schlick がやや明るい。
- **エネルギー保存**: Schlick は Smith より厳密ではない。`EnvBRDFApprox` (Karis Split-Sum) の LUT は Smith ベースなので、Schlick + IBL の混在は理論上不整合（実用上は無視可能）。

---

## 関連ドキュメント

- [[brdf_smith]] — Smith Joint Approx（標準）
- [[brdf_ggx]] — GGX NDF（D 項）
- [[brdf_fresnel_schlick]] — Schlick Fresnel（同名・別概念）
- [[../01_rendering_overview]]
