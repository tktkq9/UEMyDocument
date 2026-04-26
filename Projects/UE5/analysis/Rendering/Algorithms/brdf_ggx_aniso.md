---
name: Anisotropic GGX (S03)
description: 異方性ラフネス（接線方向 ax / 従法線方向 ay）に対応した GGX NDF。Burley 2012 Disney BRDF の拡張で、髪・拉糸金属・ヘアラインメタルなどの方向性反射を表現する。
type: project
---

# Anisotropic GGX（異方性 NDF）

- 出典 ID: **S03**（[[_source_index]]）
- UE 実装: `Engine/Shaders/Private/BRDF.ush:339` `D_GGXaniso`
- ステータス: **完了 (2026-04-26)**
- 上位: [[_algorithm_index]] / [[../01_rendering_overview]]

---

## 1. 目的

GGX NDF（[[brdf_ggx]]）の等方性 $\alpha$ を 2 軸 $\alpha_x, \alpha_y$ に拡張し、**接線フレーム（T, B, N）** に依存した方向性ハイライトを得る。
ターゲット:

- 髪（ストランドベース・カードベース）
- 拉糸金属・ヘアライン仕上げ・ブラッシュドメタル
- 木目・布の異方性（弱め）
- Substrate 系の専用シェーディングモデル（`SHADINGMODELID_HAIR`, `SHADINGMODELID_CLOTH` の一部）

---

## 2. 理論

### 2.1 異方性 NDF の定式化（Burley 2012）

$$
D(\mathbf{h}) = \frac{1}{\pi\,\alpha_x \alpha_y \left( \frac{(\mathbf{h}\cdot\mathbf{T})^2}{\alpha_x^2} + \frac{(\mathbf{h}\cdot\mathbf{B})^2}{\alpha_y^2} + (\mathbf{h}\cdot\mathbf{N})^2 \right)^2}
$$

- $\mathbf{T}$ … 接線（U 方向）
- $\mathbf{B}$ … 従法線（V 方向）
- $\alpha_x = \alpha_y$ で等方性 GGX に縮退

### 2.2 数値的に安定な等価式

直接形は `XoH²/ax² + YoH²/ay²` が小さい場合に分母が underflow する。UE は **Karis Hair Rendering 2016** の代替形を選択（`BRDF.ush:342-347`）:

```hlsl
float a2 = ax * ay;
float3 V = float3(ay * XoH, ax * YoH, a2 * NoH);
float S = dot(V, V);
return (1.0f / PI) * a2 * Square(a2 / S);
```

これは元の式と数学的に等価だが分母を `dot(V,V)` で表現するため:
- `XoH²·ay² + YoH²·ax² + (ax·ay·NoH)²` の総和を一回のドット積で計算
- 数値的に安定（FP16 でも動作）
- mad/mul のパイプラインに乗りやすい

---

## 3. UE 実装

### 3.1 BRDF.ush 内のコード

`BRDF.ush:339-352`:

```hlsl
float D_GGXaniso( float ax, float ay, float NoH, float XoH, float YoH )
{
#if 1
    // 数値安定形（Karis 2016）
    float a2 = ax * ay;
    float3 V = float3(ay * XoH, ax * YoH, a2 * NoH);
    float S = dot(V, V);
    return (1.0f / PI) * a2 * Square(a2 / S);
#else
    // 直接形（Burley 2012 そのまま）
    float d = XoH*XoH / (ax*ax) + YoH*YoH / (ay*ay) + NoH*NoH;
    return 1.0f / ( PI * ax*ay * d*d );
#endif
}
```

### 3.2 異方性パラメータ生成

UE のマテリアル系では:
- **`Anisotropy`** 入力（−1..1）と **`Roughness`** から `ax, ay` を導出
- 通常 `ax = α * (1 + Anisotropy)`, `ay = α * (1 - Anisotropy)` のような線形補間
- `Tangent` 入力で接線方向をマテリアル単位で回転

### 3.3 Visibility 項の対応

異方性 GGX の D 項とペアになる Visibility 項は **`Vis_SmithJointAniso`**（`BRDF.ush:410-415`）:

```hlsl
float Vis_SmithJointAniso(float ax, float ay, float NoV, float NoL,
                          float XoV, float XoL, float YoV, float YoL)
{
    float Vis_SmithV = NoL * length(float3(ax * XoV, ay * YoV, NoV));
    float Vis_SmithL = NoV * length(float3(ax * XoL, ay * YoL, NoL));
    return 0.5 * rcp(Vis_SmithV + Vis_SmithL);
}
```

これも 2 軸 Smith Joint のフル評価式（近似版は無い）。

---

## 4. 近似差分（理想 vs 実装）

| 項目 | 理想（Burley 2012） | UE 実装 | 補足 |
|------|--------------------|---------|------|
| NDF 形式 | 直接形（除算） | `dot(V,V)` 形（数値安定） | 等価変換 |
| Visibility | Heitz Smith Joint Aniso | 同（近似なし） | フル評価 |
| インポータンスサンプリング | Eric Heitz 2014 | `MonteCarlo.ush::ImportanceSampleVisibleGGX` | パストレーサ・IBL Pre-Filter で使用 |
| 接線フレーム入力 | 任意 | Tangent + Normal の Gram-Schmidt | 場合によって AnisoMap で回転 |

---

## 5. 主要 CVar・関連設定

異方性 BRDF 自体に CVar はないが、関連項目:

| 関連設定 | 効果 |
|---------|------|
| マテリアル `Anisotropy` 入力 | −1..1 で異方性強度（0=等方性） |
| マテリアル `Tangent` 入力 | 接線方向の回転 |
| `r.AnisotropicMaterials` | 1=有効、Substrate 非対応シェーダで使用 |
| `r.HairStrands.*` | Hair Strands 系で異方性 GGX を強制 |

---

## 6. 代替手法

| 手法 | 採用条件 | UE 実装 |
|------|---------|---------|
| **Ward Anisotropic** | 旧来（モバイル等） | 採用なし（GGX が標準） |
| **Heitz Anisotropic GGX (Visible Normals)** | パストレーサ・MC サンプリング | `MonteCarlo.ush` |
| **Marschner Hair BSDF** | Hair Strands 系 | `HairBsdf.ush`（dual-scattering 込み） |
| **Position-Free Hair (Karis 2016)** | UE 髪の改良 | `HairBsdf.ush` の Karis 派生 |

---

## 7. 参考資料

- **Burley 2012** "Physically-Based Shading at Disney" SIGGRAPH 2012 → `_papers/S03_Burley_PBS_Disney_2012.pdf`
- **Karis 2016** "Physically Based Hair Shading in Unreal" SIGGRAPH 2016 → `_papers/S03_Karis_HairShading_2016.pdf`
- **Heitz 2014** "Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs" JCGT
- 出典 ID **S03** ([[_source_index]] 参照)

---

## 8. 相談用フック（不確かなポイント）

- **`ax = α(1+Aniso), ay = α(1-Aniso)` の線形補間**は UE の慣習。Burley 元論文は $\alpha_x \cdot \alpha_y = \alpha^2$ を保つ「面積保存」マッピングを推奨しているが、UE は単純線形を選んでいる（マテリアル単位での Roughness 感覚を維持するため）。
- **`Anisotropy=−1` と `+1` の対称性**: 接線方向と従法線方向が入れ替わるだけで、数学的には同じ。マテリアル設定の符号意味は「U 軸ストレッチ vs V 軸ストレッチ」。
- **フル Smith Joint vs Approx の選択**: `Vis_SmithJointApprox` には異方性版がない。異方性ではフル Smith Joint（`Vis_SmithJointAniso`）のみ使用、近似なし。コストが約 2x。
- **Tangent 入力の不一致**: マテリアルで `Tangent` 入力が無い場合、メッシュの UV 第1チャネルから誘導された接線が使われる。UV の連続性が破綻すると異方性方向が不連続になる。

---

## 関連ドキュメント

- [[brdf_ggx]] — 等方性 GGX NDF
- [[brdf_smith]] — Smith Joint Approx（等方性版）
- [[brdf_fresnel_schlick]] — Fresnel
- [[../01_rendering_overview]]
