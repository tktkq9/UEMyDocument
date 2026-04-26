---
name: TAA (Karis 2014 — Variance Clipping + YCoCg)
description: Karis 2014 SIGGRAPH High-Quality TAA の理論と UE 実装
type: project
---

# TAA — Karis 2014 High-Quality Temporal AA

- 上位: [[_algorithm_index]]
- 関連: [[aa_tsr]] / [[brdf_split_sum]]
- 採用システム: UE5 旧 AA パス（TAA/TAAU）。新規プロジェクトは [[aa_tsr]] が標準。
- 出典:
  - **S30**: Karis 2014 SIGGRAPH "High-Quality Temporal Supersampling" (Advances in Real-Time Rendering)
  - 影響元: Yang et al. 2009 "Amortized Supersampling", Salvi 2016 "Anti-Aliasing"
  - UE 実装: `TemporalAA.usf` / `PostProcess/TemporalAA.cpp`

---

## 1. 何のためのアルゴリズムか

MSAA は forward pass のみで動作し deferred shading では使えない。SSAA は重い。空間 AA だけでは:
- サブピクセル動きでジャギー
- specular の firefly が消えない
- 髪・ワイヤーフレームのちらつき残留

### 素朴な手法の問題

- **空間 AA のみ**: サブピクセル分布をサンプルできない
- **完全ヒストリ蓄積**: ghost / smearing が顕著
- **ヒストリ無し再投影**: temporal stability ゼロ

### Karis 2014 の貢献

- **ジッター付きピクセルグリッド + 履歴アキュムレーション**で疑似 N×SSAA
- **Neighborhood Variance Clipping**（YCoCg 空間）で disocclusion / ghosting 抑制
- **Bicubic / Catmull-Rom 履歴サンプル**でリエスサンプリング鈍化を回避
- **Tone-mapped 空間でブレンド**して fireflies を抑制

---

## 2. 理論

### 2.1 ジッター付きサンプリング

毎フレーム異なる sub-pixel offset（Halton 数列等）をプロジェクション行列に適用:

```
ProjectionJittered = Translate(jitterX, jitterY) * Projection
```

UE は `View.TemporalJitterPixels` に格納。8 〜 16 フレームで 1 周し疑似 8×SSAA を達成。

### 2.2 履歴再投影

前フレーム履歴 `H_{t-1}` を motion vector で現フレーム座標へ warp:

```
uvHistory = uvCurrent - velocity
H_warp = HistoryBuffer.SampleBicubic(uvHistory)
```

bicubic / Catmull-Rom（`AA_BICUBIC` / `r.TemporalAACatmullRom`）でサンプル鈍化を抑える。線形補間だと毎フレーム 30% 程度減衰し履歴の高周波が失われる。

### 2.3 Variance Clipping（核心）

ghost を抑制するため、現フレーム近傍 3×3 サンプルの **AABB / Variance ellipsoid** を作り、履歴をその範囲にクリップ:

```hlsl
// 近傍統計
mu = mean(neighbor 3x3)
sigma = stddev(neighbor 3x3)
aabb_min = mu - gamma * sigma   // gamma ≈ 1.0
aabb_max = mu + gamma * sigma

// クリップ（クランプではなく直線 clip）
H_clipped = ClipToAABB(H_warp, aabb_min, aabb_max, mu)
```

**clip vs clamp の違い**: clamp は各成分独立に min/max → 過度に saturate。clip は履歴 → 中心 mu の直線上で AABB との交点を取る → 色相を保持。

### 2.4 YCoCg 色空間

RGB クリップだと chroma がにじむ。**YCoCg** で luma と chroma を分離:

```hlsl
Y  = 0.25*R + 0.5*G + 0.25*B
Co = 0.5*R - 0.5*B
Cg = -0.25*R + 0.5*G - 0.25*B
```

`TemporalAA.usf:82` で `AA_YCOCG 1`。luma 軸を強くクリップ、chroma 軸を緩く → 色相安定。

### 2.5 Tonemapped Blending（Firefly 抑制）

HDR 値そのままブレンドすると 1 ピクセルの 1000nit が広がる:

```
W = 1 / (1 + Luma)
A_tonemapped = A * W   // 加重平均で fireflies を抑制
B_tonemapped = B * W
result = (A_tonemapped * a + B_tonemapped * b) / (W_a * a + W_b * b)
```

`AA_TONE 1` で適用（Karis "tonemapped luma weighting"）。

### 2.6 Anti-Ghosting（動的物体対応）

`AA_DYNAMIC_ANTIGHOST` 系: motion vector がカメラ velocity と乖離する＝動的物体 → クリップ強度を上げる / 現フレーム重みを増やす。`r.TemporalAA.Quality=2` 以上で有効。

### 2.7 Responsive AA

レーザー / particle 等「履歴を残したくない」素材は stencil で `RESPONSIVE_AA` マークし、ブレンド係数を急上昇 → ほぼ現フレームのみ表示。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `PostProcess/TemporalAA.cpp` | TAA pass セットアップ、CVar、shader binding |
| `PostProcess/TemporalAA.h` | `ETAAPassConfig` / `ETAAQuality` enum |
| `TemporalAA.usf` | 本体シェーダ（PS / CS 両対応） |
| `TemporalAACommon.ush` | 共通ユーティリティ（YCoCg, clip, bicubic） |

### 3.2 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.TemporalAAFilterSize` | 1.0 | フィルタカーネル幅（小=シャープ, 大=スムーズ） |
| `r.TemporalAACatmullRom` | 0 | Catmull-Rom サンプリング |
| `r.TemporalAAPauseCorrect` | 1 | 一時停止時のヒストリ保持 |
| `r.TemporalAACurrentFrameWeight` | 0.04 | 現フレーム重み（小=安定, 大=鋭い） |
| `r.TemporalAA.Quality` | 2 | 0=最低 / 2=デフォルト / 3=ghost 抑制強 |
| `r.TemporalAA.HistoryScreenPercentage` | 100 | ヒストリ解像度倍率 |
| `r.TemporalAA.Upscaler` | 1 | 1=サードパーティ可（DLSS 等差替え） |
| `r.TemporalAA.R11G11B10History` | 1 | ヒストリ精度（1=R11G11B10, 0=fp16） |
| `r.TemporalAA.UseMobileConfig` | 0 | LDS キャッシュ無効化（モバイル） |

### 3.3 Pass Config Enum（`TemporalAA.h`）

| Config | 用途 |
|--------|------|
| `Main` | 標準 TAA（解像度等倍） |
| `MainUpsampling` | TAAU（upsampling つき） |
| `MainSuperSampling` | 2x SSAA → downsample |
| `ScreenSpace` (3) | SSR の前置 TAA |
| `LightShaft` (4) | LightShaft 専用 |
| `DiaphragmDOF` (5/6) | DOF pre-filter |

### 3.4 タイル / コンピュート

- タイルサイズ: `GTemporalAATileSizeX/Y = 8`
- LDS prefetch: 10×10（`AA_SAMPLE_CACHE_METHOD_LDS`）で帯域節約
- Mobile では `AA_MOBILE_CONFIG` で LDS 無効化（タイル GPU の FBC 優先）

---

## 4. 近似・省略の差分

| 項目 | 理想 | UE TAA | 影響 |
|------|------|-------|------|
| サンプル数 | 無限 | 8〜16 ジッタ周期 | 静止後 N フレームで完全収束 |
| Variance 推定 | 確率的 | 3x3 近傍 | エッジ近傍で過剰クリップ |
| 履歴フィルタ | 理想 sinc | bicubic / Catmull-Rom | ナイキスト超で aliasing |
| 動的物体 | per-pixel motion | velocity buffer | パーティクルなど motion 欠落で ghost |
| HDR ブレンド | 完全 reciprocal | 簡易 luma weight | 過大輝度残留 |

---

## 5. パラメータと CVar

§3.2 にまとめ済み。シャープ寄りは `r.TemporalAAFilterSize=0.7` + `r.TemporalAACatmullRom=1`。

---

## 6. 代替手法との比較

| 手法 | 機構 | UE 採用 |
|------|------|--------|
| FXAA | 空間 edge filter | ローエンド |
| MSAA | HW サンプル | Forward+ |
| **TAA (Karis 2014)** | **temporal + variance clip** | **UE5 旧標準** |
| TAAU | TAA + Upsampling | UE4.23+ |
| **TSR** | **多段機械学習風 reconstruction** | **UE5 標準** → [[aa_tsr]] |
| DLSS / FSR | NN / 解析 upscaler | プラグイン経由 |
| SMAA | edge + サブサンプル | サポート外 |

### TAA vs TSR

- TAA: 1 pass 構成、軽量、ghost 抑制は Variance Clipping のみ
- TSR: 多段（Velocity→Reject→Update→Resolve）、shading rejection / flickering 検出 / 大幅 upscale → [[aa_tsr]]

---

## 7. 参考資料

- [S30] Karis 2014 SIGGRAPH "High-Quality Temporal Supersampling"
- Salvi 2016 SIGGRAPH "Anti-Aliasing: Are We There Yet?"
- Yang et al. 2009 "Amortized Supersampling"
- 関連: [[aa_tsr]] / [[brdf_split_sum]]

---

## 8. 相談用フック

- **理解度チェック**:
  - clip と clamp の違い → 直線交点 vs 成分独立 min/max、色相保存に効く
  - YCoCg の理由 → luma と chroma を別軸でクリップ可能
  - tonemapped blending の動機 → 1 ピクセル 1000nit が周辺に広がるのを抑える
- **コード深掘り候補**:
  - `TemporalAA.usf` の `AA_YCOCG` / `ClipToAABB` 経路
  - `TemporalAACommon.ush` の bicubic 重み計算
  - `TemporalAA.cpp` の RemapPermutation（quality / pass 配線）
- **未読箇所**:
  - mobile path（FBC 優遇）と LDS path の分岐詳細
  - Responsive AA の stencil 設計
  - DOF 用 TAA pass（`TAA_PASS_CONFIG=5/6`）の CoC payload
- **次の派生**:
  - TSR → [[aa_tsr]]
  - Karis Split-Sum IBL → [[brdf_split_sum]]
  - Variance Clipping を超える Salvi 系 sample-based 手法
