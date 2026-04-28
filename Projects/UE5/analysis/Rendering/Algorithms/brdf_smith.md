---
name: Smith G2 / Joint Approximation
description: Heitz 2014 JCGT の Smith G2 整理と UE 実装（Vis_SmithJointApprox / Vis_Schlick）
type: project
---

# Smith G2 / Joint Approximation（幾何遮蔽項）

- 上位: [[_algorithm_index]]
- 関連: [[brdf_ggx]] / [[brdf_fresnel_schlick]]
- 採用システム: 全 specular 計算（GGX とペア）
- 出典 ID: **S04**（[[_source_index]]）— Heitz 2014 JCGT（Smith G2 整理 + Joint 近似提案）
- 関連: **S01** Karis 2013（Schlick 版 G の UE4 初期採用）

---

## 1. 何のためのアルゴリズムか

マイクロファセット BRDF の **幾何遮蔽項 (G, Geometry / Masking-Shadowing)**。

物理的な意味:
- **Masking**: ファセットがビューア側から隠れる確率
- **Shadowing**: ファセットが光源側から隠れる確率
- 両方を統合した結果が `G(l, v, h)`

GGX が「どの方向のファセットが何個あるか (D)」を表すのに対し、G は「**そのファセットが実際に見える/光が当たる確率**」を表す。

### 素朴な定義 (G1) vs 正しい定義 (G2)

- **G1 (separable)**: V 方向と L 方向の遮蔽を **独立** と仮定 → `G = G1(V) · G1(L)`
- **G2 (joint)**: 実際は両方向の遮蔽が **相関** している（高 Roughness で顕著） → 数学的に正しいのは joint 形式

UE は **Heitz 2014 が示した joint 形式の近似版** を採用している。

---

## 2. 理論

### 2.1 Smith G1（マスキング項）

```math
G_1(\mathbf{v}, \mathbf{h}) = \frac{2 (\mathbf{n} \cdot \mathbf{v})}{(\mathbf{n} \cdot \mathbf{v}) + \sqrt{\alpha^2 + (1 - \alpha^2)(\mathbf{n} \cdot \mathbf{v})^2}}
```

### 2.2 Smith G2 Joint（Heitz 2014）

```math
G_2(\mathbf{l}, \mathbf{v}) = \frac{2 (\mathbf{n} \cdot \mathbf{l})(\mathbf{n} \cdot \mathbf{v})}{(\mathbf{n} \cdot \mathbf{v}) \Lambda_l + (\mathbf{n} \cdot \mathbf{l}) \Lambda_v}
```

ただし `Λ` は GGX-specific な lambda 関数。

### 2.3 BRDF への組み込み（Visibility 形式）

実装では `G / (4·NoL·NoV)` をまとめて `Vis` と呼ぶ:

```math
V(\mathbf{l}, \mathbf{v}) = \frac{G(\mathbf{l}, \mathbf{v})}{4 (\mathbf{n} \cdot \mathbf{l})(\mathbf{n} \cdot \mathbf{v})}
```

最終 BRDF: `f_specular = D · V · F`

---

## 3. UE での実装

### 3.1 完全形 Smith Joint (`Vis_SmithJoint`)

`BRDF.ush:402`

```hlsl
// [Heitz 2014]
float Vis_SmithJoint(float a2, float NoV, float NoL) 
{
    float Vis_SmithV = NoL * sqrt(NoV * (NoV - NoV * a2) + a2);
    float Vis_SmithL = NoV * sqrt(NoL * (NoL - NoL * a2) + a2);
    return 0.5 * rcp(Vis_SmithV + Vis_SmithL);
}
```

### 3.2 近似版 (`Vis_SmithJointApprox`) ← UE 標準

`BRDF.ush:393`

```hlsl
// Approximation of joint Smith term for GGX
// [Heitz 2014]
float Vis_SmithJointApprox( float a2, float NoV, float NoL )
{
    float a = sqrt(a2);
    float Vis_SmithV = NoL * ( NoV * ( 1 - a ) + a );
    float Vis_SmithL = NoV * ( NoL * ( 1 - a ) + a );
    return 0.5 * rcp( Vis_SmithV + Vis_SmithL );
}
```

**完全形との違い**: `sqrt(NoV·(NoV - NoV·a2) + a2)` を `NoV·(1-a) + a` に近似（sqrt 1 回省略）。

### 3.3 異方版 (`Vis_SmithJointAniso`)

`BRDF.ush:410` — 接線基底 2 軸の `αx`, `αy` を取り、`length(float3(...))` で計算。GGX 異方とペア。

### 3.4 旧式 (`Vis_Schlick`)

`BRDF.ush:374` — UE4 初期は Karis 2013 が Schlick 1994 の G 近似を使っていた。現在は legacy 互換 / モバイル等で残るのみ。

```hlsl
float Vis_Schlick( float a2, float NoV, float NoL )
{
    float k = sqrt(a2) * 0.5;
    ...
}
```

---

## 4. 近似・省略の差分

| 項目 | 完全 G2 (S04) | UE Joint Approx | 影響 |
|------|--------------|----------------|------|
| Lambda 関数 | `sqrt(NoV·(NoV - NoV·α²) + α²)` | `NoV·(1-α) + α` | 高 Roughness で僅差（Heitz 2014 §5 で誤差グラフあり） |
| sqrt 数 | 2 回 | 0 回 | 性能向上 |
| エネルギー保存 | ◯ | ◯（実用範囲で） | なし |
| 異方性 | 可（別関数） | 同 | なし |

Heitz 2014 自身が「この近似は実用十分」と推奨している。

### Vis_Schlick との違い

- Schlick 1994 は `k = α/2` の **separable G1·G1** 形式（joint ではない）
- 高 Roughness（α > 0.5）で **エッジが暗くなりすぎる** 既知問題
- UE4 初期は採用、UE5 では Joint Approx に統一

---

## 5. パラメータと CVar

なし。シェーダ内定数のみ。

---

## 6. 代替手法との比較

| Visibility 関数 | 形式 | 採用 |
|---------------|------|------|
| `Vis_Implicit` | `0.25` 定数 | デバッグ用 |
| `Vis_Neumann` (1999) | `1 / (4·max(NoL,NoV))` | レガシー |
| `Vis_Kelemen` (2001) | `rcp(4·VoH² + ε)` | ClearCoat 第 2 層で採用 |
| `Vis_Schlick` (1994) | separable | 旧式 |
| `Vis_Smith` (1967) | full G2 | 参考実装、未使用 |
| **`Vis_SmithJoint`** (2014) | full joint | 高品質モード |
| **`Vis_SmithJointApprox`** (2014) | joint 近似 | **標準** |
| `Vis_SmithJointAniso` (2014) | 異方 joint | Anisotropy 有効時 |

---

## 7. 参考資料

- [S04] Heitz 2014, "Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs", JCGT — joint vs separable の決定的整理、Eq.99 が Joint Approx
- [S01] Karis 2013 — UE4 初期の Schlick 版 G 採用
- 関連: [[brdf_ggx]] / [[brdf_fresnel_schlick]]

---

## 8. 相談用フック

- **理解度チェック**: 
  - Joint と Separable の違いは？ → §1, §2.1-2.2
  - なぜ「Visibility」と呼ぶ？ → §2.3 `G/(4·NoL·NoV)` をひとまとめにした表記
  - Roughness 0.9 で `Vis_Schlick` が暗くなる理由 → §3.4, §6
- **コード深掘り候補**: 
  - `Vis_SmithJointApprox` の `0.5 * rcp(...)` の `0.5` の意味（理論式の `2·NoL·NoV` 分子と相殺）
  - Joint Approx と完全形の数値誤差プロット（要 `_full.md` で詳細）
- **未読箇所**: 
  - S04 §6 Importance Sampling の VNDF（Visible NDF）— Lumen のサンプリング戦略で使用
- **次の派生**: 
  - Lumen reflection サンプリング → [[lumen_reflections]]（未着手）
  - Path Tracer での G2 取り扱い → [[../RayTracing/...]]（未着手）
