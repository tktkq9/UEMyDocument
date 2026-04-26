---
name: Separable SSS (S70)
description: Jimenez & Gutierrez 2015 の Separable Subsurface Scattering。皮膚拡散プロファイルを 6 つのガウス和で近似し、水平→垂直の 2 パスで畳み込み。Burley の半解像度フォールバックとして UE5 で併存。
type: project
---

# Separable Subsurface Scattering（薄膜・半解像度モード）

- 出典 ID: **S70**（[[_source_index]]）
- UE 実装: `Engine/Shaders/Private/SeparableSSS.ush`, `Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessSubsurface.cpp`
- ステータス: **完了 (2026-04-26)**
- 上位: [[_algorithm_index]] / [[../01_rendering_overview]]

---

## 1. 目的

Subsurface Scattering プロファイルを **2D 分離可能な 1D 畳み込み**（水平 + 垂直）で近似し、Burley サンプリングより圧倒的に高速にぼかす。UE5 では:

- **半解像度モード**（`r.SSS.HalfRes=1`）の主実装
- **Transmittance 関数 `SSSSTransmittance`** で薄膜（耳・鼻翼）の光透過
- **Burley Fallback**（`r.SSS.Burley.Quality=0`）として単一フィルタ
- ライセンス上のオリジナル: GameDev コミュニティ標準実装（Jimenez & Gutierrez 2015）

---

## 2. 理論

### 2.1 拡散プロファイルのガウス和近似

皮膚の拡散カーネルを次の 6 ガウシアンの和で近似（`SeparableSSS.ush:165-170` の係数）:

$$
R(r) \approx \sum_{i=1}^{6} w_i\,e^{-r^2 / v_i}
$$

| $i$ | $w_i$ (RGB) | $v_i$ |
|----|-------------|-------|
| 1 | (0.233, 0.455, 0.649) | 0.0064 |
| 2 | (0.100, 0.336, 0.344) | 0.0484 |
| 3 | (0.118, 0.198, 0.000) | 0.187 |
| 4 | (0.113, 0.007, 0.007) | 0.567 |
| 5 | (0.358, 0.004, 0.000) | 1.99 |
| 6 | (0.078, 0.000, 0.000) | 7.41 |

これは **Transmittance 関数** で薄膜光透過を計算するときに使われる:

```hlsl
// SeparableSSS.ush:164-170
float dd = -d * d;
float3 profile = float3(0.233, 0.455, 0.649) * exp(dd / 0.0064) +
                 float3(0.1,   0.336, 0.344) * exp(dd / 0.0484) +
                 float3(0.118, 0.198, 0.0)   * exp(dd / 0.187)  +
                 float3(0.113, 0.007, 0.007) * exp(dd / 0.567)  +
                 float3(0.358, 0.004, 0.0)   * exp(dd / 1.99)   +
                 float3(0.078, 0.0,   0.0)   * exp(dd / 7.41);
```

カーネル全体は CPU 側 `ComputeMirroredBSSSKernel`（`SubsurfaceProfile.cpp:530-540`）で 3 段サイズ（13/9/6 サンプル）として事前計算される。

### 2.2 分離可能性（Separability）

ガウス関数 $G(x,y) = e^{-(x^2+y^2)/v} = e^{-x^2/v} \cdot e^{-y^2/v}$ は分離可能なので、2D 畳み込みを 1D × 1D に分解できる:

$$
(R * I)(x,y) = R_y * (R_x * I)(x,y)
$$

$O(N^2) \to O(N)$ に削減できる。皮膚プロファイルはガウシアンの**和**であって厳密には分離可能でないが、各ガウシアンは分離可能、よって**近似的**に分離可能と扱う。

### 2.3 Transmittance Function（薄膜光透過）

光が物体を貫通する場合、深度差 $d$（光源視点での厚み）から透過光を:

```
scale = 8.25 * (1.0 - translucency) / sssWidth
d = scale * |d1 - d2|   // d1=シャドウマップ深度, d2=フラグメント深度
result = profile(-d²) * saturate(0.3 + dot(L, -N))
```

として近似（`SeparableSSS.ush:137-177` `SSSSTransmittance`）。これにより耳・鼻翼の透過が表現される。

---

## 3. UE 実装（パイプライン）

```mermaid
graph LR
  Setup[SetupCS<br/>タイル分類]
  Tile[Separable Tile Buffer]
  H[Horizontal Pass<br/>SSSSBlurPS dir=1,0]
  V[Vertical Pass<br/>SSSSBlurPS dir=0,1]
  Recombine[Recombine Pass]
  Setup --> Tile
  Tile --> H
  H --> V
  V --> Recombine
```

### 3.1 SSSSBlurPS（コア畳み込み）

`SeparableSSS.ush:232-367`:

1. 中心ピクセル取得・ストレングス（0..1）読み込み
2. `finalStep = SSSScaleX / OutDepth * dir * SSSStrength` で投影スケール調整
3. カーネル長 `SSSS_N_KERNELWEIGHTCOUNT`（ガウス和の段数=13/9/6）でループ:
   - 左右対称サンプル（`Side = -1, 1`）
   - **SSSS_FOLLOW_SURFACE=1** で深度差による重み減衰
   - **BoundaryColorBleed**: 異 Profile 間の境界色滲みをチューニング
4. 重み合計で正規化、`OutColor = colorAccum / colorInvDiv`

### 3.2 サンプル数モード

`PostProcessSubsurface.cpp:79-86` `r.SSS.SampleSet`:

| 値 | サンプル数 | カーネル定義 |
|----|----------|------------|
| 0 | 6×2+1 = 13 | `SSSS_KERNEL2_SIZE=6` |
| 1 | 9×2+1 = 19 | `SSSS_KERNEL1_SIZE=9` |
| 2 | 13×2+1 = 27 | `SSSS_KERNEL0_SIZE=13`（既定） |

### 3.3 Burley との併存

`PostProcessSubsurface.cpp` のコメント（6-10 行）通り、Setup パスで **Burley タイル** と **Separable タイル** を別バッファに振り分け、Indirect Dispatch で並列実行する。`r.SSS.HalfRes.ForceSeparable=1` のとき Half-Res は強制的に Separable のみ。

---

## 4. 近似差分（理想 vs 実装）

| 項目 | 理想（Jimenez 2015） | UE 実装 | 補足 |
|------|---------------------|---------|------|
| カーネル分離性 | 厳密にはガウス和は不可 | 各成分を分離して総和 | 視覚的に許容 |
| 深度感度 | 体積追跡 | `SSSS_FOLLOW_SURFACE` Bilateral | 急峻な深度差で漏れ |
| プロファイルカスタマイズ | 任意関数 | 6 ガウシアン固定（皮膚向け） | 皮膚以外は MC でフィッティング |
| Boundary Color Bleed | 単一マテリアル前提 | プロファイル ID 比較で抑制 | 異 Profile 接合部の対処 |
| Transmittance | 単純 LUT 参照 | scale=8.25 マジック値 | 厚み計算の経験則 |
| Subpixel | サンプル無し | 1px 以下のスキップ（`Subpixel.Threshold`） | 性能最適化 |

---

## 5. 主要 CVar

[[sss_burley]] と共通の `r.SSS.*` を使用。Separable 固有のものは:

| CVar | 既定 | 効果 |
|------|------|------|
| `r.SSS.SampleSet` | 2 | 0=13サンプル, 1=19, 2=27（既定） |
| `r.SSS.HalfRes.ForceSeparable` | 1 | HalfRes で Burley を強制的に Separable へ |
| `r.SSS.Burley.Quality` | 1 | 0=Burley → Separable フォールバック |
| `r.SSS.Filter` | 1 | 0=point、1=bilinear（カーネルサンプリング時） |
| `r.SSS.Checkerboard.NeighborSSSValidation` | 0 | 境界色漏れ抑制（追加コスト） |

---

## 6. 代替手法

| 手法 | 採用条件 | UE 実装 |
|------|---------|---------|
| **Burley Normalized SSS** | フル解像度・高品質 | [[sss_burley]] |
| **Pre-Integrated Skin** | モバイル | （簡易 Forward） |
| **Path Tracer** | オフライン | リファレンス比較 |
| **Texture Space SSS** | UV ストリーミング・キャラ単位 | 採用なし（実装コストが高い） |

---

## 7. 参考資料

- **Jimenez, Gutierrez 2015** "Separable Subsurface Scattering" CGF 2015 → `_papers/S70_Jimenez_SeparableSSS_2015.pdf`
- **Jimenez 2010** "Screen-Space Perceptual Rendering of Human Skin" → 旧フォーマット（`SSSSBlurPS` の祖先）
- iryoku.com/sssss/, iryoku.com/translucency/ — 著者公開リソース
- 出典 ID **S70** ([[_source_index]] 参照)

---

## 8. 相談用フック（不確かなポイント）

- **6 ガウシアンの係数**（`0.233, 0.455, 0.649` …）は Jimenez 論文の Caucasian skin 計測値。皮膚以外（蝋・葉・大理石）に流用しても合わないので、UE 内では Subsurface Profile アセットの `FalloffColor`/`SurfaceAlbedo` から `ComputeMirroredBSSSKernel` で動的フィッティング（`SubsurfaceProfile.cpp:530`）。
- **`scale = 8.25 * (1 - translucency) / sssWidth`** の `8.25` は Jimenez のデモコードのマジック値。UE のシーンスケール（cm）依存。
- **`SSSS_FOLLOW_SURFACE` の `12000/400000` 重み**（`SeparableSSS.ush:335`）も経験則。
- **Separable Approximation の限界**: 高曲率・薄膜では分離性が破綻し、Reference Path Tracer と比較するとリングアーティファクトが出る場合がある（耳の輪郭で顕著）。Burley 推奨。
- **Transmittance 経路は SeparableSSS の Reflectance パスとは独立**で、両モードを切り替えても薄膜光透過関数は共通利用される（Burley 経由でも `SSSSTransmittance` を呼ぶ）。

---

## 関連ドキュメント

- [[sss_burley]] — Burley Normalized SSS（フル解像度・主実装）
- [[shadow_pcf_pcss]] — Shadow Map → Transmittance の入力源
- [[../01_rendering_overview]]
