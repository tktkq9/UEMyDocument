---
name: Filmic Tone Mapping (Hable / Reinhard)
description: 軽量 Filmic Tone Curve の理論と UE 実装（HableToneMap / SimpleReinhardToneMap / FilmToneMap）
type: project
---

# Filmic Tone Mapping — Hable / Reinhard / UE Film Curve

- 上位: [[_algorithm_index]]
- 関連: [[tone_aces]] / [[post_bloom]]
- 採用システム: HDR ディスプレイ非対応 / モバイル / 軽量プロファイル / 旧プロジェクト互換
- 出典 ID: **S51**（[[_source_index]]）— Reinhard et al. 2002 (SIGGRAPH)
- 影響元: Hable 2010 GDC "Uncharted 2: HDR Lighting"（Filmic Curve）/ Hable 2017 Blog "Piecewise Power Curves"
- UE 実装: `TonemapCommon.ush` / `PostProcessTonemap.cpp`

---

## 1. 何のためのアルゴリズムか

[[tone_aces]] が高品質だが計算重い。Hable/Reinhard 系は **5〜6 演算で済む S 字曲線** で:

- HDR (0..∞) → SDR (0..1) に圧縮
- 中間階調を線形に保ちつつ、暗部・ハイライトを smooth 飽和

シネマ ACES が要らない用途（モバイル、UE3 互換 look、編集者好み）で歴史的に重宝された。

### 物理的位置付け

写真フィルムの D-log E 曲線（Density vs Log Exposure）の電子近似。フィルム露光の toe（黒つぶれ）と shoulder（白飛び）を S 字で再現することで「映画ぽい」見た目になる。

---

## 2. 理論

### 2.1 Reinhard（2002 オリジナル）

```
L_d = L / (1 + L)
```

- L: シーン輝度
- L_d: ディスプレイ輝度 (∈ [0, 1))
- ∞ → 1 へ漸近、中間で線形比 ≒ 1

#### Extended Reinhard（白点指定）

```
L_d = L * (1 + L / Lwhite²) / (1 + L)
```

`Lwhite` 以上の輝度を白に焼き付ける。視認限界を任意設定可能。

#### UE 実装

`TonemapCommon.ush:636-639`:
```hlsl
float3 SimpleReinhardToneMap(float3 LinearColor)
{
    return LinearColor / (1.0f + LinearColor);
}
```

ベース Reinhard のみ。Diagnostic / 比較用途。

### 2.2 Hable Filmic Curve（Uncharted 2 系）

6 定数 (A〜F) のラショナル多項式:

```
F(x) = ((x*(A*x + C*B) + D*E) / (x*(A*x + B) + D*F)) - E/F
```

`TonemapCommon.ush:613-623` の係数:
```hlsl
A = 0.15  // Shoulder Strength
B = 0.50  // Linear Strength
C = 0.10  // Linear Angle
D = 0.20  // Toe Strength
E = 0.02  // Toe Numerator
F = 0.30  // Toe Denominator
```

#### White Point Normalization

```
W = 11.2
WhiteScale = 1 / F(W)
ExposureBias = 2.0
result = F(linear * ExposureBias) * WhiteScale
```

`F(W)` で割ることで W 入力 → 1.0 出力に正規化（白がきっちり 1.0 になる）。

### 2.3 UE Film Curve（FilmToneMap、ACES 派生 Hable approx）

`TonemapCommon.ush:105-220` の `FilmToneMap()` は ACES の RRT 風 glow + red modifier を持ちつつ Hable 風の S 字を **ACEScg ワーキング空間で log10 ドメイン** で評価する hybrid:

1. sRGB → AP0/AP1 変換
2. RRT_GLOW (gain 0.05, mid 0.08) を `sigmoid_shaper((sat-0.4)/0.2)` で適用
3. Red Modifier: hue 0° 中心 ±67.5° に対し `0.82` でスケール
4. AP1 → log10
5. **三段曲線**: Toe / Straight / Shoulder（[[tone_aces]] §2.4 と同式）
6. Pre-desaturate 0.96, Post-desaturate 0.93

このため **「UE Film Curve = Filmic と ACES の橋渡し」**。完全 ACES より軽く、生 Hable より色再現が良い。

### 2.4 Slope / Toe / Shoulder の意味

```
FilmSlope     : 線形部の勾配（コントラスト）
FilmToe       : 暗部の S カーブ強度（黒締まり）
FilmShoulder  : 明部の S カーブ強度（白の柔らかさ）
FilmBlackClip : 黒のベースオフセット（持ち上げ量）
FilmWhiteClip : 白のクリップ余裕
```

UE プリセット（`TonemapCommon.ush:68-97` のコメント）:

| Preset | Slope | Toe | Shoulder | BlackClip | WhiteClip |
|--------|-------|-----|----------|-----------|-----------|
| Uncharted | 0.63 | 0.55 | 0.47 | 0 | 0.01 |
| HP (High Punch) | 0.65 | 0.63 | 0.45 | 0 | 0 |
| Legacy (UE3) | 0.98 | 0.30 | 0.22 | 0 | 0.025 |
| ACES (UE4 デフォ) | 0.91 | 0.53 | 0.23 | 0 | 0.035 |

### 2.5 FilmToneMapInverse

`TonemapCommon.ush:222-252` で逆変換も提供:
- ガラス越しに HDR 値を取り戻したい / SSR で線形に戻したい場合に使用
- 解析的 inverse なので O(1) コスト

### 2.6 Tonemap 後の Encoding

Filmic 後は **gamma 変換 → 8bit 量子化 → backbuffer**:

```hlsl
half3 TonemapAndGammaCorrect(half3 LinearColor)
{
    LinearColor = max(LinearColor, 0);
    half3 GammaColor = pow(LinearColor, InverseGamma.x);  // 通常 1/2.2
    return saturate(GammaColor);
}
```

`TonemapCommon.ush:53-65`。SDR ターゲットでは backbuffer が sRGB-formatted なら GPU 自動 sRGB エンコード→ pow を省略可。

---

## 3. UE での実装

### 3.1 主要関数

| 関数 | ファイル:行 | 用途 |
|------|------------|------|
| `SimpleReinhardToneMap` | `TonemapCommon.ush:636` | Diagnostic / Reference 比較 |
| `HableToneMapCurve` | `TonemapCommon.ush:613` | 6 係数 Hable 曲線 |
| `HableToneMap` | `TonemapCommon.ush:625` | White scale + ExposureBias 込み |
| `FilmToneMap` | `TonemapCommon.ush:105` | UE デフォルト（ACES 派生） |
| `FilmToneMapInverse` | `TonemapCommon.ush:222` | Inverse |
| `TonemapAndGammaCorrect` | `TonemapCommon.ush:53` | Gamma 適用 + 量子化用 saturate |

### 3.2 PostProcess Settings

| 名前 | 用途 |
|------|------|
| `FilmSlope` | 線形部勾配 (`FilmToneMap` の `FilmSlope`) |
| `FilmToe` | toe 強度 |
| `FilmShoulder` | shoulder 強度 |
| `FilmBlackClip` | 黒オフセット |
| `FilmWhiteClip` | 白クリップ |
| `FilmGrainIntensity` | film grain |

これらは `FPostProcessSettings` から `FilmSlope` 等 uniform に渡される（`TonemapCommon.ush:99-103`）。

### 3.3 Permutation

`PostProcessTonemap.cpp:84-98` で permutation:
```c
FTonemapperGammaOnlyDim     // ガンマだけで Tonemap 完全省略
FTonemapperLocalExposureDim // ローカル露出 0/1/2
FTonemapperSharpenDim       // シャープネス
FTonemapperFilmGrainDim     // film grain
FTonemapperMsaaDim          // iOS Metal MSAA pre-resolve
FTonemapperAlphaChannelDim  // alpha 透過対応
```

`USE_GAMMA_ONLY=1` パスは Filmic を完全バイパス（DEBUG 用 / extreme low-end）。

### 3.4 LUT との関係

通常パスは Tonemap + ColorGrading + ガンマを **3D LUT に焼く**（`PostProcessCombineLUTs.cpp`）→ Tonemap shader は `LUT.Sample()` のみ。Filmic 関数本体は LUT 構築時に走るだけで、毎フレームではない。

`r.TonemapperFilm=0` で生 Filmic 直呼び（LUT 経由しない）も可能。

---

## 4. 近似・省略の差分

| 項目 | 公式 Hable | UE Filmic（FilmToneMap） | 影響 |
|------|-----------|------------------------|------|
| 色空間 | sRGB 直 | ACEScg AP1 経由 | hue 一貫性 ↑ |
| Glow / Red modifier | なし | あり（ACES 流） | ハイライトの色相維持 |
| Hue 維持 | per-channel curve | desaturate + log | 純色の安定性 ↑ |
| Inverse | 解析逆なし | `FilmToneMapInverse` 提供 | 逆計算可能 |
| パラメータ | A〜F 6 個固定 | Slope/Toe/Shoulder 自由 | アーティスト調整可 |

---

## 5. パラメータと CVar

§3.2 と PostProcessVolume の Film Tone セクション。Tone Mapper 完全バイパスは `r.TonemapperFilm=0` + `r.TonemapperGamma>0`。

---

## 6. 代替手法との比較

| 手法 | 形式 | コスト | UE 採用 |
|------|------|-------|--------|
| **SimpleReinhard** | `L/(1+L)` | 1 div | **diagnostic** |
| Reinhard Extended | `L*(1+L/W²)/(1+L)` | 数演算 | 自前実装可 |
| **Hable Filmic** | 6 係数有理多項式 | 数十演算 | **`HableToneMap` 提供** |
| Hable 2017 Piecewise Power | toe/linear/shoulder 独立指数 | 数十演算 | 未採用 |
| **UE FilmToneMap** | log S 字 + ACES 周辺 | 〜100 演算 | **デフォルト Tone Curve** |
| **ACES v1.3 RRT+ODT** | log S 字 + glow + ODT 完全版 | 数百演算 | **HDR 出力時** → [[tone_aces]] |
| AgX | Sigmoid + Lab hue 補正 | 中 | プラグイン |
| Lottes 2016 | `x^a/(x^a+b)` GTI | 軽量 | 未採用 |

### UE 内での使い分け

| シーン | 推奨 |
|-------|------|
| SDR sRGB ディスプレイ通常 | **`FilmToneMap` (デフォルト)** |
| HDR10 / Dolby Vision | **ACES** → [[tone_aces]] |
| モバイル軽量 | `HableToneMap` または LUT |
| デバッグ / 比較 | `SimpleReinhardToneMap` |
| 完全 raw 出力 (LUT 焼き等) | `r.TonemapperGamma>0` でバイパス |

---

## 7. 参考資料

- [S51] Reinhard et al. 2002 "Photographic Tone Reproduction for Digital Images" SIGGRAPH
- Hable 2010 GDC "Uncharted 2: HDR Lighting"
- Hable 2017 Blog "Filmic Tonemapping with Piecewise Power Curves" (http://filmicworlds.com/)
- Lottes 2016 GDC "Advanced Techniques and Optimization of HDR Color Pipelines"
- 関連: [[tone_aces]] / [[post_bloom]]

---

## 8. 相談用フック

- **理解度チェック**:
  - Reinhard `L/(1+L)` の限界 → 中間階調が圧縮されすぎ、純色 hue が崩れる
  - Hable Filmic の white scale 理由 → `F(W)` で割って白を 1.0 に正規化
  - UE FilmToneMap が ACES の派生である理由 → glow + red modifier + log S 字を引き継ぎつつ調整可能化
- **コード深掘り候補**:
  - `TonemapCommon.ush:105-220` `FilmToneMap` の三段曲線実装
  - `TonemapCommon.ush:613-623` `HableToneMapCurve` の係数
  - `TonemapCommon.ush:222-252` `FilmToneMapInverse` の解析逆
- **未読箇所**:
  - LUT 構築時の Filmic 評価コスト（`PostProcessCombineLUTs.cpp`）
  - `r.TonemapperFilm=0` 時のシェーダパス差分
  - Mobile Tonemapper（`PostProcessTonemap.cpp` mobile 版）の差分
- **次の派生**:
  - HDR/シネマ用 ACES → [[tone_aces]]
  - Bloom が Tonemap 前段にある関係 → [[post_bloom]]
  - Auto Exposure → 別途解析
