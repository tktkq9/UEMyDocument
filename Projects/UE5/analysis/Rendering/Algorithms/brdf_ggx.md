---
name: GGX / Trowbridge-Reitz NDF
description: Walter 2007 の GGX NDF と UE 実装（D_GGX / D_GGXaniso）+ Disney Roughness パラメータ化
type: project
---

# GGX / Trowbridge-Reitz NDF（法線分布関数）

- 上位: [[_algorithm_index]]
- 関連既存ドキュメント: [[../BasePass/Details/...]]（未着手） / [[../DeferredLighting/Details/...]]（未着手）
- 採用システム: BasePass, DeferredLighting, Lumen, RayTracing, Substrate（全シェーディングモデルの specular 項）
- 出典 ID: **S02**（[[_source_index]]）— Walter et al. 2007（GGX NDF 原典）
- 関連: **S03** Burley 2012（Roughness パラメータ化と GGX 採用根拠）
- 関連: **S01** Karis 2013（UE 採用と最適化）

---

## 1. 何のためのアルゴリズムか

**マイクロファセット理論** に基づく specular BRDF の核心となる **法線分布関数 (NDF, Normal Distribution Function)**。

表面を「微小な完全鏡面ファセット（microfacet）の集合」と仮定した時、ハーフベクトル `H` の方向を向いているファセットの**統計的な密度**を表す関数 `D(H)` が NDF。

### 素朴な手法の問題

- **Phong / Blinn-Phong**: 物理的根拠なし、エネルギー保存が崩れる、ハイライト形状が硬い
- **Beckmann NDF**（Cook-Torrance 1981）: ハイライト裾野の減衰が速すぎ、現実の素材より「シャープ」に見える
- **GGX**: 裾野が長く（long-tail）、現実の素材（特に金属）の観測に最も近い

---

## 2. 理論

### 2.1 GGX の数式（Walter 2007 Eq.33）

```math
D_{\text{GGX}}(\mathbf{h}) = \frac{\alpha^2}{\pi \, \big( (\mathbf{n} \cdot \mathbf{h})^2 (\alpha^2 - 1) + 1 \big)^2}
```

- `n`: 表面法線（マクロ）
- `h`: ハーフベクトル `(L+V)/||L+V||`
- `α`: 表面粗さパラメータ (`alpha`)

### 2.2 Roughness と α の関係（Burley 2012）

ユーザー入力の **Roughness** を直接 `α` として使うのではなく、

```math
\alpha = \mathrm{Roughness}^2
```

と二乗してから使う。これは Burley が Disney で採用し、UE4 (Karis 2013) も踏襲したパラメータ化:

- **理由**: アーティストにとって Roughness スライダの **視覚的線形性** を保つため。Roughness = 0.1 と 0.2 で見た目の差が一定になる
- **物理的意味**: GGX の数式では `α` が小さいと急激にハイライトが鋭くなる（非線形）。`α = R²` にすると低 Roughness 領域での解像度が増える

### 2.3 等方 / 異方

| 種類 | 数式 | パラメータ |
|------|------|----------|
| 等方 | `α² / (π · d²)` | `α` 1 つ |
| 異方 | `1 / (π · αx·αy · d²)` （d は接線基底依存） | `αx`, `αy` 2 つ |

---

## 3. UE での実装

### 3.1 等方 GGX (`D_GGX`)

`Engine/Shaders/Private/BRDF.ush:331`

```hlsl
// GGX / Trowbridge-Reitz
// [Walter et al. 2007, "Microfacet models for refraction through rough surfaces"]
float D_GGX( float a2, float NoH )
{
    float d = ( NoH * a2 - NoH ) * NoH + 1;  // 2 mad
    return a2 / ( PI*d*d );                  // 4 mul, 1 rcp
}
```

| 引数 | 意味 |
|------|------|
| `a2` | `α² = Roughness⁴`（呼び出し側で前計算） |
| `NoH` | `n · h`（コサイン） |

### 3.2 理論式との対応

理論式の分母:

```
( NoH² · (α² - 1) + 1 )²
```

UE 実装の分母 (`d² = ((NoH·a2 - NoH)·NoH + 1)²`) を展開:

```
(NoH·a2 - NoH) · NoH + 1
= NoH²·a2 - NoH² + 1
= NoH² · (a2 - 1) + 1   ← 理論式と一致
```

つまり **数学的には完全一致**。違いは MAD 命令にまとめた最適化のみ（コメントの "2 mad" が示す）。

### 3.3 異方 GGX (`D_GGXaniso`)

`BRDF.ush:339`

```hlsl
// [Burley 2012, "Physically-Based Shading at Disney"]
float D_GGXaniso( float ax, float ay, float NoH, float XoH, float YoH )
{
#if 1
    float a2 = ax * ay;
    float3 V = float3(ay * XoH, ax * YoH, a2 * NoH);
    float S = dot(V, V);
    return (1.0f / PI) * a2 * Square(a2 / S);
#else
    float d = XoH*XoH / (ax*ax) + YoH*YoH / (ay*ay) + NoH*NoH;
    return 1.0f / ( PI * ax*ay * d*d );
#endif
}
```

`#if 1` の方は **数学的には等価で、除算を 1 回に減らすベクトル形式** に書き換えたもの。

`XoH`, `YoH` は接線基底（タンジェント・バイタンジェント）と H の内積。

### 3.4 Roughness → α² 変換の場所

呼び出し側（例: `ShadingModels.ush`）で `a2 = Pow4(Roughness)` (= `(R²)²`) を計算してから `D_GGX` に渡す。BRDF.ush 内には変換コードはない（呼び出し責務）。

---

## 4. 近似・省略の差分

| 項目 | 理論 (Walter 2007) | UE 実装 | 影響 |
|------|------------------|--------|------|
| NDF 数式 | Eq.33 そのまま | 完全一致（MAD 最適化のみ） | なし |
| `α` 定義 | `α = Roughness`（論文時点） | `α = Roughness²` | アーティスト用パラメータ化 |
| ゼロ除算保護 | なし | `MinRoughness = 0.001` を上流でクランプ | 真の鏡面（α=0）は表現不可、影響軽微 |
| 異方の表現 | 接線基底依存（同じ） | 同左 | なし |

---

## 5. パラメータと CVar

| CVar / パラメータ | デフォルト | 範囲 | 効果 |
|-----------------|----------|------|------|
| `Material.Roughness` | 0.5 | 0.0-1.0 | アーティスト入力の Roughness |
| `MinRoughness` (定数) | 0.001 | - | `BRDF.ush:600` `RoughnessXY` 系で使用 |
| `r.Material.RoughnessOverride` | -1 | 0-1 | デバッグ用、全マテリアル一括上書き |

---

## 6. 代替手法との比較

| NDF | 数式特徴 | UE での採用 |
|-----|---------|----------|
| Phong（指数 cos） | `(n·h)^N`、物理根拠なし | レガシー（Mobile Forward の一部のみ） |
| Blinn-Phong | Phong の改良版 | 同上 |
| Beckmann | `exp((NoH²-1)/(α²·NoH²)) / (π·α²·NoH⁴)` | `D_Beckmann` (`BRDF.ush:323`) で実装あり、未使用 |
| **GGX (Trowbridge-Reitz)** | 長い裾野、観測一致度最高 | **標準採用**、`D_GGX` |
| GGX 異方 | タンジェント基底 2 軸 | Material 側で「Anisotropy」有効時 `D_GGXaniso` |
| Inverse GGX (布) | `D_InvGGX` (`BRDF.ush:705`) | Cloth ShadingModel のみ |

### Beckmann vs GGX の見た目差分

- 同じ Roughness で **GGX の方がハイライト周囲がぼんやり広がる**（=現実的）
- Beckmann は中央のピークが鋭く、エッジが急に消える
- 高 Roughness（金属の艶消し等）では GGX の優位性が顕著

---

## 7. 参考資料

- [S01] Karis 2013, "Real Shading in Unreal Engine 4" — slide 11 で UE4 の GGX 採用宣言
- [S02] Walter et al. 2007, "Microfacet Models for Refraction through Rough Surfaces" — GGX (Eq.33) 原典
- [S03] Burley 2012, "Physically-Based Shading at Disney" — `α = Roughness²` 採用根拠
- 関連: [[brdf_smith]] — GGX とペアで使う Smith G2 幾何項
- 関連: [[brdf_split_sum]] — IBL での GGX 前畳み込み

---

## 8. 相談用フック

セッション中に Claude が即提示できる「相談の起点」:

- **理解度チェック**: 
  - なぜ Beckmann ではなく GGX？ → §6 比較表 + S03 §5.4
  - なぜ `α = Roughness²` か？ → §2.2、視覚的線形性の話
  - `Pow4(Roughness)` は何階乗？ → `α² = (R²)² = R⁴`
- **コード深掘り候補**: 
  - `D_GGX` 内の MAD 最適化展開（§3.2 の式変形）
  - `D_GGXaniso` の `#if 1` ベクトル形式の利点（除算 1 回 vs 2 回）
- **未読箇所**: 
  - S02 §4 全反射の取り扱い（屈折を含む完全 BSDF）
  - S03 §5.5 ClearCoat の二重 BRDF
- **次の派生**: 
  - Substrate での GGX 拡張 → [[../Substrate/...]]（未着手）
  - VSM サンプリングと NDF の関係 → [[vsm_overview]]

未解決事項（`_source_index.md` の質問表に転記済み）:
- `α = Roughness²` の Disney 内部経緯（S03 だけでは詳細不明）
- 高 Roughness 時の Energy Conservation の崩れ（Kulla-Conty 補正の必要性）
