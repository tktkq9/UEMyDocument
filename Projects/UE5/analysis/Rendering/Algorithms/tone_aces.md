---
name: ACES Tone Mapping (Academy Color Encoding System)
description: ACES の理論（IDT / RRT / ODT）と UE 実装（v1.3 と v2.0 ACES 2 切替）
type: project
---

# ACES — Academy Color Encoding System

- 上位: [[_algorithm_index]]
- 関連: [[tone_filmic]] / [[post_bloom]]
- 採用システム: シネマ向け HDR 出力の事実上の業界標準。UE5 のデフォルト Tone Curve も ACES 派生
- 出典 ID: **S50**（[[_source_index]]）— ACES v1.3 / v2.0 (Academy Color Encoding Specification, AMPAS)
- 関連規格: SMPTE ST.2065-1 (ACES2065-1) / SMPTE ST.2084 (PQ EOTF)
- UE 実装: `ACES/ACES_v1.3.ush` / `ACES/ACESOutputTransform.ush` / `ACES/ACESDisplayEncoding.ush` / `TonemapCommon.ush`

---

## 1. 何のためのアルゴリズムか

HDR シーン（リニア値、明るさ範囲 [0, ∞)）を、各種出力デバイス（sRGB モニタ、Rec.709 TV、HDR10 1000nit、Dolby Vision、シネマ DCI-P3 など）に「同じ見た目」で送り出すための **業界標準カラー パイプライン**。

### 解きたい問題

- HDR レンダ結果の輝度ダイナミックレンジ ∞ → ディスプレイ最大 100〜10000 nit に圧縮
- 色域（ACEScg AP1 / sRGB / DCI-P3 / Rec.2020）変換と gamut 圧縮
- ハイライトでの不自然な hue shift 抑止（純赤が黄色ぽく飛ぶ等）
- マスタリングと配信デバイスの分離（同一 master → 多数の出力）

### 単純 Reinhard / Hable では足りない理由

- hue 一貫性なし → ハイライトで色相がずれる
- HDR モニタへの ST2084 (PQ) エンコード未対応
- 色域広い（Rec.2020）デバイスへ送れない

---

## 2. 理論

### 2.1 ACES パイプライン全体

```
Camera RAW
   │ IDT (Input Device Transform) ← 各カメラ専用
   ▼
ACES2065-1 (AP0 wide gamut, scene-linear)
   │ RRT (Reference Rendering Transform) ← 1 個固定
   ▼
OCES (Output Color Encoding Specification, scene-referred)
   │ ODT (Output Device Transform) ← 各出力デバイス専用
   ▼
表示デバイス
```

UE5 のレンダリングは IDT を経ずに **ACEScg (AP1)** で生成 → そのまま RRT+ODT に流し込む。

### 2.2 色空間

| 略称 | 用途 | 特徴 |
|------|------|------|
| AP0 | ACES2065-1 マスタ | xyY 座標で人間視認可能域を完全包含、ネガティブ値多用 |
| AP1 | ACEScg ワーキング | レンダリング用、AP0 の subset、原色は negative 抑制 |
| sRGB | sRGB / Rec.709 出力 | ガンマ 2.2 近似 |
| DCI-P3 | デジタルシネマ | sRGB より広い |
| Rec.2020 | UHD HDR | ほぼ視認域カバー |

### 2.3 RRT（Reference Rendering Transform）

シーン参照値（HDR 線形）→ シーン参照 OCES への "look" 適用。要素:

1. **Glow Module**: 暗部彩度低下時に微小グロー追加
   - `RRT_GLOW_GAIN=0.05`, `RRT_GLOW_MID=0.08`
2. **Red Modifier**: 純赤彩度ダウン（ハイライトで「焼け飛び」防止）
   - `RRT_RED_SCALE=0.82`, `RRT_RED_PIVOT=0.03`, `RRT_RED_HUE=0`, `RRT_RED_WIDTH=135`
3. **Pre-desaturate**: AP1 に降りて 0.96 で desat
4. **Tonescale**: log-domain で toe / straight / shoulder の三段曲線（後述）
5. **Post-desaturate**: 0.93 で再 desat

UE 実装は `TonemapCommon.ush:105-220` の `FilmToneMap()` がこれを **ACES 互換 Hable approximated** で再構成（高速、固定品質）。

### 2.4 Tonescale（v1.3 旧式 / v2.0 新式）

#### v1.3（UE のデフォルト、`USE_ACES_2 == 0`）

S 字曲線を log10 ドメインで toe / straight / shoulder に分け、`FilmSlope` / `FilmToe` / `FilmShoulder` / `FilmBlackClip` / `FilmWhiteClip` で形状制御:

```
StraightColor   = FilmSlope * (LogColor + StraightMatch)
ToeColor        = -FilmBlackClip + (2*ToeScale) / (1 + exp(-2*FilmSlope/ToeScale * (LogColor - ToeMatch)))
ShoulderColor   = (1+FilmWhiteClip) - (2*ShoulderScale) / (1 + exp( 2*FilmSlope/ShoulderScale * (LogColor - ShoulderMatch)))
```

`TonemapCommon.ush:201-213`。プリセット例:

| Preset | Slope | Toe | Shoulder | BlackClip | WhiteClip |
|--------|-------|-----|----------|-----------|-----------|
| Uncharted | 0.63 | 0.55 | 0.47 | 0 | 0.01 |
| Legacy (UE3) | 0.98 | 0.30 | 0.22 | 0 | 0.025 |
| ACES (UE4 デフォ) | 0.91 | 0.53 | 0.23 | 0 | 0.035 |

#### v2.0（`USE_ACES_2 == 1`、`ACESOutputTransform.ush`）

ACES 2.0 (2024) は CAM 系（Hellwig 2022）の知覚色域モデルへ全面刷新:
- **JMh** 空間（Lightness J / Colorfulness M / Hue h）で gamut compression
- ターゲット PeakLuminance (`peakLuminance`) ごとに `init_ODTParams()` で curve 計算
- LUT 3 枚（`ReachTable` / `GamutTable` / `GammaTable`）を事前焼き込み → CPU 構築 / GPU サンプル

`outputTransform_fwd()` が forward 変換、`outputTransform_inv()` が逆変換（HDR mastering 用）。

### 2.5 ODT（Output Device Transform）

`UE_ACES_EOTF` で出力 EOTF を切替（`TonemapCommon.ush:305-310`）:

```
0: ST-2084 (PQ)        ← HDR10
1: BT.1886             ← Rec.709 / Rec.2020 SDR
2: sRGB                ← sRGB ディスプレイ
3: gamma 2.6           ← DCI シネマ
4: linear              ← LUT 焼き / scRGB
5: HLG                 ← BBC/NHK HDR
```

PQ 系では `LinearToNitsScale = 100.0` (`TonemapCommon.ush:50`) で正規化（1.0 == 100 nit を意味する）。

### 2.6 Gamut Compression

ACES 2.0 の白眉。出力色域より外の彩度を smooth 圧縮:

- `compressionThreshold = 0.75` 以上の彩度を tanh ベースで圧縮
- `smoothCusps=0.12`, `smoothM=0.27`, `cuspMidBlend=1.3` などで cusp 形状調整
- LUT 化されているので shader cost は texture fetch のみ

v1.3 では `compressColor()`（`ACES_v1.3.ush`）+ `GamutCompression` ブレンド係数。

### 2.7 White Limiting / Scaling

`ACESDisplayEncoding.ush:39-65` の `white_limiting`:
- 出力 RGB 各チャネルを `0` 〜 `peakLuminance/referenceLuminance` でクランプ
- `apply_white_scale` でカラーキャストせず最大チャネルを 1.0 へ正規化
  - 理由: HDR モニタの校正白と作業白が違うと unequal RGB で純白を作る → 1ch 先飽和で hue ずれ → これを防ぐ

### 2.8 出力フォーマット enum（`TonemapCommon.ush:23-32`）

```c
TONEMAPPER_OUTPUT_sRGB                = 0
TONEMAPPER_OUTPUT_Rec709              = 1
TONEMAPPER_OUTPUT_ExplicitGammaMapping = 2
TONEMAPPER_OUTPUT_ACES1000nitST2084   = 3   // HDR10 1000nit
TONEMAPPER_OUTPUT_ACES2000nitST2084   = 4   // HDR10 2000nit
TONEMAPPER_OUTPUT_ACES1000nitScRGB    = 5   // Windows HDR 1000
TONEMAPPER_OUTPUT_ACES2000nitScRGB    = 6   // Windows HDR 2000
TONEMAPPER_OUTPUT_LinearEXR           = 7   // mastering write
TONEMAPPER_OUTPUT_NoToneCurve         = 8   // diagnostic
TONEMAPPER_OUTPUT_WithToneCurve       = 9
```

`HDRHelper.h` で BackBuffer pixel format から自動選択。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `PostProcessTonemap.cpp` | Tonemap pass セットアップ、CVar 一式、permutation 構築 |
| `PostProcessTonemap.h` | API |
| `TonemapCommon.ush` | `FilmToneMap` / `HableToneMap` / `SimpleReinhardToneMap` / `ACESOutputTransform` ラッパ |
| `ACES/ACES_v1.3.ush` | 旧 ACES（compressColor / outputTransform） |
| `ACES/ACESCommon.ush` | 色変換マトリクス・xyY 関連 |
| `ACES/ACESOutputTransform.ush` | v2.0 outputTransform_fwd / inv、ODTParams 構造体 |
| `ACES/ACESDisplayEncoding.ush` | v2.0 white_limiting / clamp |
| `ACES/ACESTonescale.ush` | v2.0 tone curve LUT 構築 |
| `ACES/ACESUtilities.ush` | v2.0 hue table / wrap helper |
| `PostProcessCombineLUTs.cpp/.usf` | カラーグレーディング LUT 焼き |
| `HDRHelper.h/.cpp` | 出力デバイス detection / format 選択 |

### 3.2 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.Tonemapper.Sharpen` | -1 | -1=PostProcessVolume、0=off、1=full |
| `r.TonemapperGamma` | 0 | 0=デフォルト、>0=固定 gamma で上書き |
| `r.Gamma` | 1.0 | 出力 gamma scale |
| `r.HDR.UsePostProcessACES` | 1 | HDR 出力時の ACES 適用 ON/OFF |
| `r.HDR.Display.OutputDevice` | 0 | OS HDR API 経由で自動 |
| `r.HDR.Display.ColorGamut` | 0 | 0=Rec.709, 1=DCI-P3, 2=Rec.2020 |
| `r.HDR.Display.MaxLuminance` | 0 | 0=自動検出 |
| `USE_ACES_2` (shader define) | 0 | 1=ACES 2.0 パス採用、要 LUT 構築コスト |

### 3.3 PostProcess Settings（`FPostProcessSettings`）

- **Film Slope / Toe / Shoulder / BlackClip / WhiteClip**: §2.4 v1.3 曲線パラメータ（Editor で各種プリセット）
- **Scene Color Tint / Saturation / Contrast / Gamma / Gain / Offset**: ACES 前段の color grading 群（CombineLUTs にベイク）
- **Color Grading LUT** (`ColorGradingLUT`): ユーザー LUT を ACES と組み合わせ可能

### 3.4 LUT Combine 経路

実行時は毎フレーム ODT を計算するのは重いため:

1. CombineLUTs Pass で 32×32×32 (or 16×16×16) の **3D LUT** を構築
   - 入力: 線形 SceneColor → 出力: encoded display value
   - ACES tonecurve + gamut + ODT + color grading + ガンマ までベイク
2. Tonemap Pass はピクセルごとに `LUT.Sample(SceneColor)` だけ → ALU 大幅節約
3. パラメータ変更時のみ rebuild（`bDirty` フラグ）

### 3.5 ACES 2.0 の LUT 3 枚

`ACESOutputTransform.ush:95-97` の `ODTParams::reach_table / gamut_table / gamma_table`:

- **ReachTable**: hue ごとの最大 chroma reach
- **GamutTable**: hue ごとの cusp 位置 (J, M)
- **GammaTable**: hue ごとの lower hull gamma

CPU 側で peakLuminance に応じて 1D table（hue 360 段）を構築 → R32F 1D テクスチャでアップロード。

### 3.6 PQ / scRGB Encoding

```
SceneLinear (nits) → / referenceLuminance(100) → ACESOutputTransform → 
  PQ EOTF inv (ST2084) → 10bit / 12bit code → HDR10 backbuffer
```

scRGB の場合は 16-bit float リニア (D65 white = 80 nit が 1.0)。`UE_SCRGB_WHITE_NITS = 80.0` (`TonemapCommon.ush:322`)。

---

## 4. 近似・省略の差分

| 項目 | 公式 ACES v1.3 | UE 実装 | 影響 |
|------|---------------|--------|------|
| RRT 内部式 | log-Sigmoid 5 次セグメント | Hable approximated S 字 | 高輝度で 1〜2 % 差、視覚的にはほぼ同じ |
| Gamut Compress | per-pixel iterative | LUT 1 段ベイク | エッジで 1 LSB 程度の差 |
| Glow Module | フル | 簡略化（goldenglow 寄り） | 暗部のグロー量微差 |
| ODT EOTF | デバイス毎 | 6 種固定 | カスタム HLG 設定不可 |
| ACES 2.0 採用 | 2024 リリース | `USE_ACES_2` 任意 | デフォルト OFF、experimental |

---

## 5. パラメータと CVar

§3.2 の CVar 表 + PostProcessVolume の Film Tone セクションを併用。HDR モニタは OS 検出に任せ、SDR では sRGB 固定。

---

## 6. 代替手法との比較

| 手法 | 出典 | 特徴 | UE 採用 |
|------|------|------|--------|
| Reinhard | Reinhard 2002 | `L/(1+L)` 単純、hue 維持なし | `SimpleReinhardToneMap` (`TonemapCommon.ush:636`) |
| Hable Uncharted 2 | Hable 2010 | 6 係数固定 S 字 | `HableToneMap` (`TonemapCommon.ush:625`) |
| Reinhard Extended (Lwhite) | Reinhard 2002 | 白点指定可 | 直接実装なし |
| Filmic SMPTE 2084 | 各社 | カスタム S 字 + PQ | UE 旧 Filmic |
| **ACES v1.3 RRT+ODT** | AMPAS 2014 | log-S 字 + glow + red modifier | **UE5 デフォルト Tone Curve** |
| **ACES v2.0** | AMPAS 2024 | CAM JMh + LUT | **`USE_ACES_2` ON 時** |
| AgX | Sobotka 2023 | sigmoid + 知覚 hue 補正 | プラグイン (Marmoset 採用) |
| Lottes 2016 | Lottes/AMD | `x^a / (x^a + b)` 形 GTI | 未採用 |

### Filmic との関係

UE の **Film Tone Curve は ACES の派生** で、`FilmToneMap` 内部で `sRGB → AP1 → AP0 → glow → red modifier → log S字 → AP1 出力` を行う。完全な ACES ODT を毎ピクセルで走らせる代わりの軽量実装。完璧な ACES が欲しい場合は `USE_ACES_2=1` 経路または LUT 焼きで CombineLUTs を通す。

---

## 7. 参考資料

- [S50] ACES v1.3 / v2.0 仕様 (AMPAS, https://acescentral.com/)
- ACES TB-2014-004 "Color Conversion for ACES2065-1"
- Reinhard 2002 "Photographic Tone Reproduction for Digital Images"
- Hable 2010 "Filmic Tonemapping Operators" GDC
- Hellwig & Fairchild 2022 "Brightness, Lightness, Colorfulness, and Chroma..." (ACES 2.0 採用 CAM)
- 関連: [[tone_filmic]] / [[post_bloom]]

---

## 8. 相談用フック

- **理解度チェック**:
  - RRT と ODT の役割分担 → 1 個の RRT で OCES に上げ、デバイス毎の ODT で出力空間に降ろす
  - AP0 と AP1 の使い分け → AP0 はマスタ用、AP1 はレンダ用
  - ACES 2.0 が CAM ベースになった理由 → hue 一貫性向上、HDR モニタの広い peakLum に対応
- **コード深掘り候補**:
  - `TonemapCommon.ush:105-220` `FilmToneMap` の log S 字構築
  - `ACESDisplayEncoding.ush:39-65` `white_limiting` の hue 維持機構
  - `ACESOutputTransform.ush:59-98` `ODTParams` のフィールド
- **未読箇所**:
  - CombineLUTs での ACES 焼き手順（CPU/GPU 分担）
  - `USE_ACES_2` 切替時のシェーダ permutation 影響
  - HDR10 vs Dolby Vision の出力差分処理
- **次の派生**:
  - 軽量 Filmic → [[tone_filmic]]
  - Bloom 後段で ACES に渡す前提 → [[post_bloom]]
  - Color Grading LUT → CombineLUTs 別途解析
