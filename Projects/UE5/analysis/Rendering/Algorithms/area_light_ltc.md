---
name: Linearly Transformed Cosines for Area Lights (S08)
description: Heitz 2016 の LTC 近似。GGX BSDF を線形変換された Cosine 分布として表現することで、矩形・円盤エリアライトの解析的積分を可能にする。UE の Rect Light 実装の中核。
type: project
---

# Linearly Transformed Cosines（LTC）

- 出典 ID: **S08**（[[_source_index]]）
- UE 実装: `Engine/Shaders/Private/RectLight.ush:335` `RectApproxLTC`（旧記載 `:334` はコメント行 / 関数定義は 335 行）, `:403` `RectGGXApproxLTC`, `Engine/Shaders/Private/RectLightIntegrate.ush`
- ステータス: **完了 (2026-04-27)**
- 上位: [[_algorithm_index]] / [[../01_rendering_overview]]

---

## 1. 目的

Rect Light（および Disk Light）の物理ベース照明を **解析的に**積分する。MC サンプリングなしで、GGX BSDF と多角形エリアライトの積を一発で求められる。

採用先:
- Rect Light（UE の主要エリア光源）
- Disk Light の Specular 寄与（極限での Rect 退化として）
- Area Light Sheen / ClearCoat（5.4+）
- Substrate Multi-Lobe Area Light

---

## 2. 理論

### 2.1 LTC の中心アイデア（Heitz 2016）

任意の球面分布 $D(\omega)$ を **基準 Cosine 分布** $D_0(\omega) = \cos\theta/\pi$ の **3×3 線形変換** で近似:

$$
D(\omega) \approx \frac{D_0(\mathbf{M}^{-1}\omega)}{|\det\mathbf{M}|\,(\mathbf{M}^{-1}\omega)_z^3}
$$

Cosine 分布の多角形積分は **解析解**（[Lambert 1760, Baum 1989]）が存在する:

$$
\int_P \cos\theta\,d\omega = \sum_{i=1}^{n}\arccos(\langle\mathbf{p}_i, \mathbf{p}_{i+1}\rangle)\,\langle\mathbf{n}, \mathbf{p}_i \times \mathbf{p}_{i+1}\rangle
$$

これにより GGX BSDF + 多角形ライトの積分を 4 頂点なら 4 個の arccos と外積で評価できる。

### 2.2 LTC LUT の構造

UE は事前計算 2D テクスチャを保持:
- 軸: $(N\!\cdot\!V, \alpha)$
- 内容: $\mathbf{M}^{-1}$ の **4 つの非ゼロ成分**（$\mathbf{M}$ は対称的に削減形に配置）
  - 通常 `(a, b, c, d)` 4 チャネル
  - 対角ブロック構造で 9 自由度を 4 自由度に圧縮
- 元論文の順序を UE 形式に合わせて再フィット

### 2.3 PolygonIrradiance（解析積分）

Heitz の参考実装に近い形:

```
for each edge (i, i+1):
    cosTheta = saturate(dot(P[i], P[i+1]))
    theta = acos(cosTheta)
    edge += theta * normalize(cross(P[i], P[i+1])).z
return edge / (2*PI)
```

UE では `MonteCarlo.ush` 系の `PolygonIrradiance` 関数で同等処理。

---

## 3. UE 実装

### 3.1 GetRectLTC_GGX（LUT 取得）

`RectLight.ush:300〜` で:
- LUT サンプル `(a, b, c, d)` を `PreIntegratedGFSampler` から取得
- 3×3 行列 `LTC` と `InvLTC` を構築（対称構造を利用）
- IrradianceScale を SheenLTC.z 等から取得

### 3.2 RectApproxLTC（コア積分）

`RectLight.ush:335-400`（旧記載 `:334` はコメント行 / 関数定義は 335 行）:

```hlsl
// [Heitz et al. 2016, "Real-Time Polygonal-Light Shading with Linearly Transformed Cosines"]
float3 RectApproxLTC(FRectLTC In, half3 N, float3 V, FRect Rect, ...)
{
    // Tangent space 構築
    float3 T1 = normalize(V - N * dot(N, V));
    float3 T2 = cross(N, T1);
    float3x3 TangentBasis = float3x3(T1, T2, N);

    In.LTC = mul(In.LTC, TangentBasis);
    In.InvLTC = mul(transpose(TangentBasis), In.InvLTC);

    // Rect の 4 頂点を LTC 空間へ変換
    float3 Poly[4];
    Poly[0..3] = mul(In.LTC, Rect.Origin ± Axis*Extent);

    // 解析積分
    float3 L = PolygonIrradiance(Poly);

    // 球面近似 + Cosine 重み補正
    float SinAlphaSqr = length(L);
    float NoL = SphereHorizonCosWrap(L.z, SinAlphaSqr);
    float Irradiance = SinAlphaSqr * NoL;

    // ワールド空間に戻す + ソーステクスチャサンプル
    L = mul(In.InvLTC, normalize(L));
    return SampleSourceTexture(L, Rect, ...) * Irradiance * In.IrradianceScale;
}
```

ポイント:
- **`SphereHorizonCosWrap`** で水平線下の寄与をラップ（球面近似でグレージング修正）
- **Mean Light Direction** $L$ を逆変換でワールド空間に戻し、Rect Source Texture（ライトテクスチャ・スポット模様）をサンプル
- LTC 行列は **N と V から構築した接線基底** に合わせて回転される

### 3.3 RectGGXApproxLTC（GGX 専用エントリ）

`RectLight.ush:403-412`:

```hlsl
float3 RectGGXApproxLTC( float Roughness, float3 SpecularColor, ... )
{
    const float NoV = saturate( abs( dot(N, V) ) + 1e-5 );
    const FRectLTC LTC = GetRectLTC_GGX(Roughness, SpecularColor, NoV);
    return RectApproxLTC(LTC, N, V, Rect, ...);
}
```

ラフネスから LTC LUT をサンプルし、共通のコア関数へ渡す薄いラッパ。

### 3.4 統合パス

`RectLightIntegrate.ush` の `IntegrateBxDF` で:
- Diffuse Lobe: `RectApproxLTC` （Lambert LTC = ほぼ単位行列）
- Specular Lobe: `RectGGXApproxLTC`
- Sheen / ClearCoat: 各 Lobe 用の専用 LTC LUT

---

## 4. 近似差分（理想 vs 実装）

| 項目 | 理想 GroundTruth | UE LTC | 補足 |
|------|----------------|--------|------|
| GGX BSDF 表現 | 厳密 | 線形変換 Cosine 近似 | 高 Roughness で精度落ち |
| 多角形積分 | MC（数千サンプル） | 解析解（4 arccos） | 圧倒的高速 |
| 異方性 | 不対応（標準 LTC） | 等方性のみ | 異方性は別 LUT 必要 |
| エネルギー保存 | 厳密 | LUT に IrradianceScale 込み | 多重散乱補正 |
| グレージング | 厳密 | `SphereHorizonCosWrap` で近似 | Disk Light 退化 |
| ライトテクスチャ | 各ピクセルでサンプル | Mean Light Direction でサンプル | テクスチャ模様の高周波が失われる |

LTC は Heitz らによって**最大誤差 4%**程度に抑えられている（元論文）。Rect Light は MC 32 サンプル相当の品質。

---

## 5. 主要 CVar

| CVar | 既定 | 効果 |
|------|------|------|
| `r.RectLightAtlas.MaxResolution` | 1024 | RectLight ソーステクスチャ Atlas 最大解像度 |
| `r.RectLightAtlas.SamplingMips` | 1 | Atlas のミップ生成（ぼかしたサンプリング用） |
| `r.AreaLightShadows` | 1 | エリアライトのシャドウ有効化 |
| `r.AreaLights.SoftShadows` | 1 | ソフトシャドウ（VSM や Distance Field） |
| `r.RayTracing.RectLight.NumSamples` | 8 | RT 経路での MC サンプル数（LTC ではなく RT 時） |
| `r.MobileShading.AreaLights` | 0 | Mobile での Area Light 対応 |

---

## 6. 代替手法

| 手法 | 採用条件 | UE 実装 |
|------|---------|---------|
| **MC Sampling (32 サンプル)** | RT 経路 | `RectLightIntegrate.ush:99-119` |
| **Karis Representative Point** | 旧モデル（5.0 前） | 採用なし（LTC で置換済み） |
| **Spherical Harmonics** | Sky Light / 大エリア | 別系統（IrradianceVolume） |
| **Disk LTC**（特化版） | 円盤光源 | Rect の 8 頂点近似 + LTC |

UE 5.x では Rect Light のリアルタイム経路は完全に LTC ベース。RT 経路のみ MC が選択肢。

---

## 7. 参考資料

- **Heitz, Dupuy, Hill, Hill 2016** "Real-Time Polygonal-Light Shading with Linearly Transformed Cosines" SIGGRAPH 2016 → `_papers/S08_Heitz_LTC_2016.pdf`
- **Heitz et al. 2017** "Combining Analytic Direct Illumination and Stochastic Shadows" → ソフトシャドウ統合
- Eric Heitz の LTC リファレンス実装: https://github.com/selfshadow/ltc_code
- 出典 ID **S08** ([[_source_index]] 参照)

---

## 8. 相談用フック（不確かなポイント）

- **LUT の 4 自由度圧縮**: $\mathbf{M}^{-1}$ は本来 9 自由度だが、Heitz 2016 では対称性で 4 個に削減される。UE のチャネル割り当て `(a, b, c, d)` の物理的意味は元論文と完全には一致しない（UE 用に再フィットされている）。
- **`SphereHorizonCosWrap` の挙動**: `RectLight.ush:378` で球面近似による水平線ラッピングを適用。低 Roughness でグレージング光源があると微妙なリングアーティファクトが出る場合あり。
- **Mean Light Direction でテクスチャサンプル**: ライトテクスチャ模様の高周波が「平均化」されて失われる。模様の解像感が必要な場合は MC 経路（RT）推奨。
- **異方性 GGX + Rect Light**: LTC は等方性のみ対応。マテリアルが異方性の場合、現状 UE では等方性へのフォールバックが起こる（Substrate でも同様）。
- **Sheen LTC との関係**: `IrradianceScale = SheenLTC.z` の流用は Substrate 系の構造再利用。Sheen と GGX で別 LUT だが共通ヘルパを使う。
- **Disk Light**: Rect を「8 頂点近似」で扱う実装と「Disk 専用 LTC」の切替がコード中で複数経路ある（`AreaLightCommon.ush`）。どちらが採用されるかはシェーディングモデル依存。

---

## 関連ドキュメント

- [[brdf_split_sum]] — Karis Split-Sum（点光源 IBL の対応物）
- [[brdf_ggx]] / [[brdf_ggx_aniso]] — NDF
- [[brdf_smith]] — Smith Joint Approx
- [[../01_rendering_overview]]
