---
name: Lambert Diffuse (古典)
description: Lambert (1760) の理想拡散モデル。BRDF = albedo / π の最も基本的な拡散項。UE では Disney Diffuse の代替として、低スペック・モバイル・Path Tracer のリファレンスで使用される。
type: project
---

# Lambert Diffuse（古典拡散モデル）

- 出典 ID: **古典**（Lambert 1760, Lambert's Cosine Law）
- UE 実装: `Engine/Shaders/Private/BRDF.ush:177` `Diffuse_Lambert`
- ステータス: **完了 (2026-04-26)**
- 上位: [[_algorithm_index]] / [[../01_rendering_overview]]

---

## 1. 目的

最もシンプルな物理ベース拡散モデル。**等方的に光を拡散**する理想ランバート面の BRDF を提供する。
採用先:

- Mobile / 低スペック経路（Disney Diffuse のコスト削減版）
- Path Tracer のリファレンス（GroundTruth 比較用）
- 古典 PBR ワークフロー（旧プロジェクト互換）
- Cloth・Hair の Diffuse Component の一部

新規プロジェクトでは [[brdf_disney_diffuse]] が標準。

---

## 2. 理論

### 2.1 Lambert's Cosine Law

理想拡散面では、入射光のすべての方向に等方的に再放射される。BRDF として:

$$
f_d(\mathbf{l}, \mathbf{v}) = \frac{\rho}{\pi}
$$

- $\rho$ … Albedo（0..1 の RGB）
- $\pi$ … 半球積分 $\int_\Omega \cos\theta\,d\omega = \pi$ で正規化

レンダリング方程式の Diffuse 寄与:

$$
L_d = \frac{\rho}{\pi}\,L_i\,(\mathbf{n}\cdot\mathbf{l})_+
$$

cosine 重み（$N\!\cdot\!L$）はライトループ側で適用するため、BRDF 関数自体は $\rho/\pi$ のみを返す。

### 2.2 等方性の含意

- 視線方向 $\mathbf{v}$ に依存しない（V 不変性）
- 表面の見た目が観察方向によらず一定（"matte"）
- Roughness パラメータを持たない（常に完全粗い面）
- レトロ反射（光源方向への戻り光増強）を表現できない

---

## 3. UE 実装

### 3.1 BRDF.ush

`BRDF.ush:177-180`:

```hlsl
half3 Diffuse_Lambert( half3 DiffuseColor )
{
    return DiffuseColor * (1 / PI);
}
```

- 入力: `DiffuseColor`（Albedo）
- 出力: BRDF 値 $\rho/\pi$
- コスト: 3 mul（最小、`(1/PI)` はコンパイル時定数）

half 精度でもオーバーフローしない最も安価な拡散関数。

### 3.2 切替マクロ

UE では `MaterialTemplate.ush` 等の `#if MATERIAL_*` 定数で拡散モデルを切替:

| マクロ | 採用関数 | 文脈 |
|--------|---------|------|
| `DIFFUSE_TYPE_LAMBERT` | `Diffuse_Lambert` | Mobile・古典経路 |
| `DIFFUSE_TYPE_BURLEY` | `Diffuse_Burley` | 標準 Deferred ([[brdf_disney_diffuse]]) |
| `DIFFUSE_TYPE_OREN_NAYAR` | `Diffuse_OrenNayar` | 旧モバイル・布 |
| `DIFFUSE_TYPE_GOTANDA` | `Diffuse_Gotanda` | 実験・コンソール検証 |
| `DIFFUSE_TYPE_EON` | `Diffuse_EON` | 5.5+ エネルギー保存ラフディフューズ |

### 3.3 Path Tracer Reference

`PathTracer.usf` の Diffuse Lobe では、CG リファレンス比較のため `Diffuse_Lambert` が選択肢として用意されている（`r.PathTracer.DiffuseModel=0`）。GroundTruth とのデルタ計測に使う。

---

## 4. 近似差分（理想 vs 実装）

| 項目 | 理想 Lambert | UE 実装 | 補足 |
|------|------------|---------|------|
| 形式 | $\rho/\pi$ | 同じ | 完全一致 |
| 表面粗さ依存 | なし | なし | 仕様通り |
| エネルギー保存 | $\int_\Omega f_d \cos\theta\,d\omega = \rho \le 1$ | 同 | 厳密 |
| レトロ反射 | なし | なし | 表現不可 |
| Fresnel との結合 | なし | コーディングで `(1 - F)` を掛けて結合 | エネルギー保存のため |

物理的に最もシンプルで「正しい」拡散モデル。粗い面の理想化として機能。

---

## 5. 主要 CVar・選択

| CVar / マクロ | 効果 |
|------|------|
| `r.Material.DiffuseModel` | Diffuse モデルの切替（プラットフォーム別） |
| `r.PathTracer.DiffuseModel` | パストレーサ用切替（0=Lambert, 1=Burley, 2=…） |
| Mobile 経路 | コンパイル時マクロで強制 Lambert |
| Cloth Shading | `ClothShading.ush` 内で Diffuse Component に Lambert 採用 |

CVar 切替よりもコンパイルマクロで決定されることが多い。

---

## 6. 代替手法

| 手法 | 採用条件 | UE 関数 |
|------|---------|---------|
| **Disney Diffuse (Burley 2012)** | 標準 Deferred | [[brdf_disney_diffuse]] / `Diffuse_Burley` |
| **Oren-Nayar** | 粗い拡散面（粘土・コンクリート） | `Diffuse_OrenNayar` |
| **Gotanda 2014** | 実験的高品質 | `Diffuse_Gotanda` |
| **EON (Portsmouth 2025)** | 5.5+ エネルギー保存 | `Diffuse_EON` |
| **GGX_Rough Diffuse** | 高 Roughness 整合 | `Diffuse_GGX_Rough` |

---

## 7. 参考資料

- **Lambert, J.H. 1760** "Photometria" Latin treatise — 概念の出典
- **Pharr, Jakob, Humphreys 2016** "Physically Based Rendering" 3rd ed., Chap 8.2 — 教科書定義
- **Karis 2013** "Real Shading in Unreal Engine 4" SIGGRAPH 2013 — UE が Lambert から Burley に移行した経緯
- 出典 ID **古典**（[[_source_index]] では論文記載なし）

---

## 8. 相談用フック（不確かなポイント）

- **`half3` 戻り値**: 関数シグネチャは `half3` だが、呼び出し側が float に昇格させる場合がほとんど。half 精度で問題が出る場面は少ないが、HDR 強度のライトでは float 化推奨。
- **`(1/PI)` のコンパイル時定数化**: `PI` がマクロ／const として展開されるため、`(1/PI)` は単一 mul に最適化される。`Pow5` 等のヘルパーと組み合わせる場合のみ注意。
- **エネルギー保存と Fresnel との結合**: Lambert 単体ではエネルギー保存。Specular（Schlick Fresnel）と組み合わせる際、ライト評価側で `Diffuse * (1 - F)` を掛けて Fresnel 反射ぶんを差し引くのが慣習。Burley は元々 Fresnel 込みなのに対し、Lambert は素のため、結合方法が異なる。
- **モバイル経路の選択ロジック**: 実機では feature level で自動切替。`MobileBasePassPixelShader.usf` の `#if MOBILE_USE_BURLEY` 等を確認しないと確定しない。

---

## 関連ドキュメント

- [[brdf_disney_diffuse]] — Disney Diffuse / Burley 2012（標準）
- [[brdf_fresnel_schlick]] — Fresnel との結合
- [[brdf_ggx]] / [[brdf_smith]] — Specular 側
- [[../01_rendering_overview]]
