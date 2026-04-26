---
name: Bloom (Mip-pyramid Convolution)
description: Mip-pyramid Gaussian Bloom の理論と UE 実装（BloomSetup → DownsampleChain → GaussianBlur 加算）
type: project
---

# Bloom — Mip-pyramid Gaussian Convolution

- 上位: [[_algorithm_index]]
- 関連: [[post_bloom_fft]] / [[tone_aces]] / [[tone_filmic]]
- 採用システム: 標準 Bloom（高速、quality 1〜5）。FFT Bloom と相互排他
- 出典:
  - **S52**: Mittring 2007 / Kawase 2003 GDC "Frame Buffer Postprocessing Effects in DOUBLE-S.T.E.A.L (Wreckless)"（mip-pyramid bloom 系の出典）
  - 古典理論: Gaussian カーネル分離可能性 + 多段 down/up sample
  - UE 実装: `PostProcessBloomSetup.cpp` / `PostProcessBloom.usf` / `PostProcessWeightedSampleSum.cpp/.usf`

---

## 1. 何のためのアルゴリズムか

HDR シーンで「眩しい光源」が眼やレンズで滲む現象（眼内散乱・レンズ収差・センサ ブルーム）の再現:

- 太陽 / 強い光源周辺の白いハロー
- ネオンや夜景の光の "にじみ"
- HDR 値（>1.0）が SDR モニタで見える兆候の代用

Tonemap で潰される高輝度のヘッドルームを **可視化する役割** も大きい。

### 素朴な Gaussian Blur との違い

巨大カーネル（数百〜数千 px）の単発 Gaussian は重すぎる:
- 半径 R の 2D カーネル: O(R²) per pixel
- フル解像度で R=256 → 65,536 サンプル / px → 不可能

**Mip pyramid Gaussian**:
- 半径 R を「N 段 down-sample × 各段で固定半径ぶれ」で **対数時間** に分解
- 各段カーネル小さい（5〜13 tap）
- 段数 6〜7 で R=数百 px 等価ブルームを得る

---

## 2. 理論

### 2.1 Gaussian の分離可能性

```
G(x, y) = G(x) × G(y)
```

2D Gaussian は X と Y の独立 1D の積 → **Pass を 2 回（X, Y）に分解** で O(R²) → O(2R)。

### 2.2 Mip Pyramid 分解

ガウシアンの自己畳み込み性質:
```
G(σ₁) * G(σ₂) = G(√(σ₁² + σ₂²))
```

連続ダウンサンプル + 同程度の小カーネルブラーで **大半径 ガウシアンに収束**:
- 1/2 解像度で σ₁ → 等価 σ = 2σ₁
- 1/4 解像度で σ₂ → 等価 σ = 4σ₂
- ... 6 段で 64σ₆ 相当

UE は **3〜6 段の Bloom Stage** を別々の `Bloom1Size`〜`Bloom6Size` で持ち、 各段の出力を **加算合成** することで「複数スケールで違う色味のブルーム」を作れる:

```
Final = Σ_i  Tint_i × Stage_i
```

### 2.3 BloomSetup（前処理パス）

`PostProcessBloom.usf:37-67` `BloomSetupCommon`:

```hlsl
SceneColor = Sample(InputTexture) * View.OneOverPreExposure;

#if USE_LOCAL_EXPOSURE
    // Local Exposure 補正をかけてから bloom 抽出
    LocalExposure = CalculateLocalExposure(...);
    SceneColor.rgb *= LocalExposure;
#endif

#if USE_THRESHOLD
    Lum = Luminance(SceneColor) * ExposureScale;
    BloomLuminance = Lum - BloomThreshold;
    BloomAmount = saturate(BloomLuminance * 0.5);
#else
    BloomAmount = 1.0;
#endif

return BloomAmount * SceneColor * View.PreExposure;
```

- **Pre-exposure 正規化**: `OneOverPreExposure` で線形値に戻す → 計算 → `PreExposure` で再スケール
- **閾値 (`BloomThreshold`)**: HDR ピーク以上のみ抽出（古典手法、UE5 はデフォルト無効＝物理 bloom）
- **Local Exposure 連動**: `r.LocalExposure` 経路で BloomSetup 段でも局所補正
- **Threshold 無効＝物理ベース**: シーン全体に対し弱く均等に bloom がかかる。Tonemap 後の自然な眩しさ表現

### 2.4 DownsampleChain

`SceneDownsampleChain` が事前に SceneColor を **6 段ダウンサンプル** したテクスチャを保持（`PostProcessDownsample.cpp` の責務）:

```
Stage 0: Full
Stage 1: 1/2
Stage 2: 1/4
Stage 3: 1/8   ← Bloom Q1/Q2 はここから
Stage 4: 1/16  ← Bloom Q3
Stage 5: 1/32  ← Bloom Q4
Stage 6: 1/64  ← Bloom Q5
```

`BloomQualityToSceneDownsampleStage = {-, 3, 3, 4, 5, 6}` (`PostProcessBloomSetup.cpp:235-243`)。 Quality 上げると最深 stage 増 → カーネルがより広範囲。

### 2.5 GaussianBlur Pass（加算ベース）

各 stage で `AddGaussianBlurPass` を呼び:

```c
PassInputs.Filter   = SceneDownsampleChain->GetTexture(SourceIndex);  // 入力 mip
PassInputs.Additive = PassOutputs;                                     // 前 stage の累積
PassInputs.CrossCenterWeight = CrossCenterWeight;                      // r.GaussianBloom.Cross
PassInputs.KernelSizePercent = BloomStage.Size * Settings.BloomSizeScale;
PassInputs.TintColor = BloomStage.Tint * TintScale;
```

`PostProcessWeightedSampleSum.cpp` 内で **X 方向 1D Gaussian** → **Y 方向 1D Gaussian** + 前 stage の累積を加算。`KernelSizePercent` は viewport 比 で半径指定（解像度独立）。

### 2.6 Cross Bloom（疑似アナモルフィック）

`r.GaussianBloom.Cross` (`PostProcessBloomSetup.cpp:18-26`):
- `>0`: 中心サンプルに重み追加 → "+" 字状のクロスフレア（DSLR の絞り回折ぽい）
- `<0`: X 方向のみ強調 → アナモルフィックレンズの横長フレア
- `0`: 通常 Gaussian

軽量（カーネル中心 weight 増やすだけ）で見栄え向上。

### 2.7 Tint と Stage 数

`Settings.Bloom1Tint`〜`Bloom6Tint` を各 stage に独立に与えられる:
- Stage 1（小半径）: 黄色っぽい
- Stage 6（大半径）: 青っぽい
で「色収差付きハロー」を演出。`TintScale = (1/MAX) * BloomIntensity` で正規化（合計が `BloomIntensity` に揃う）。

### 2.8 PreExposure と物理整合

UE5 では **PreExposure** 機構で SceneColor を `View.PreExposure` 倍してから格納（fp16 オーバーフロー回避）。Bloom Setup で `OneOverPreExposure` を掛けて素の HDR 値に戻し、出力時に再 `PreExposure` を掛ける → bloom が **シーンの実輝度** に比例。Tonemap 後 ACES に渡るので物理整合性保たれる。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `PostProcessBloomSetup.cpp` | BloomSetup pass + AddGaussianBloomPasses（mip-pyramid 構築） |
| `PostProcessBloomSetup.h` | API（`FBloomSetupInputs`, `AddBloomSetupPass`, `AddGaussianBloomPasses`） |
| `PostProcessBloom.usf` | BloomSetupPS / BloomSetupCS シェーダ |
| `PostProcessDownsample.cpp/.usf` | SceneDownsampleChain 構築 |
| `PostProcessWeightedSampleSum.cpp/.usf` | 1D Gaussian + Additive blend |
| `PostProcessFFTBloom.cpp` | FFT Bloom 経路（[[post_bloom_fft]]） |

### 3.2 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.BloomQuality` | 5 | 0=Disabled / 1〜5=stage 数 |
| `r.GaussianBloom.Cross` | 0 | クロスフレア / アナモルフィック |
| `r.Bloom.ScreenPercentage` | 100 | FFT 経路用 |
| `r.Bloom.AsyncCompute` | 1 | FFT 経路の非同期 |
| `r.Bloom.CacheKernel` | 1 | FFT 経路 spectral kernel cache |
| `r.Bloom.WarnKernelResolution` | 1 | FFT kernel 解像度警告 |

### 3.3 PostProcess Settings

| 名前 | 用途 |
|------|------|
| `BloomMethod` | 0=Standard (mip-pyramid), 1=FFT Convolution |
| `BloomIntensity` | 全体強度 |
| `BloomThreshold` | 閾値（-1 で物理 bloom = 全画素） |
| `BloomSizeScale` | 全 stage 共通サイズスケール |
| `Bloom1Size`〜`Bloom6Size` | 各 stage の半径% |
| `Bloom1Tint`〜`Bloom6Tint` | 各 stage の色味 |
| `BloomGaussianIntensity` | 標準経路の追加スケール |
| `BloomConvolutionTexture` | FFT 経路用カーネル画像 |

### 3.4 EBloomQuality

```
EBloomQuality::Disabled  // 0
EBloomQuality::Q1〜Q5    // 1〜5
EBloomQuality::MAX       // 6 (sentinel)
```

Q1/Q2 は最深 stage 3、Q5 は最深 stage 6 → quality 上げると広範囲 + コスト ↑。

### 3.5 Local Exposure 連動

`USE_LOCAL_EXPOSURE` permutation で `LumBilateralGrid` / `BlurredLogLum` を取り、 `CalculateLocalExposure()` で局所トーンマッピングを bloom 入力にも適用。Tonemap pass と同じ意図で、**局所的に明るい場所** に bloom を強める。

### 3.6 Compute / Pixel 両対応

`bIsComputePass = View.bUseComputePasses` で CS / PS 分岐:
- **CS**: 8×8 タイルで処理、`RWOutputTexture` UAV 書き込み
- **PS**: full-screen quad、`RenderTargets[0]` 経由

---

## 4. 近似・省略の差分

| 項目 | 物理 PSF / FFT Bloom | UE Mip-pyramid | 影響 |
|------|---------------------|---------------|------|
| カーネル形状 | 任意（円・星・回折縞） | Gaussian のみ | リアルなアパーチャ反映なし |
| 半径 | 連続値 | 6 段の離散和 | カーブが Gaussian の和に縛られる |
| 計算量 | O(N log N) FFT | O(N) per stage × 6 = O(N) | mip-pyramid 圧勝 |
| 異方性 | 任意 | Cross/Anamorphic 軽量近似のみ | 本物の bokeh 不可 |
| 色域 | wavelength 別可 | RGB のみ tint | 物理的 chromatic aberration なし |

---

## 5. パラメータと CVar

§3.2 / 3.3 にまとめ済み。FFT へ切替は `BloomMethod=Convolution` (`PostProcessBloomSetup.h:IsFFTBloomEnabled` 参照)。

---

## 6. 代替手法との比較

| 手法 | 形状 | 速度 | UE 採用 |
|------|------|------|--------|
| Single Gaussian Blur | 1 サイズ | O(R²)→O(2R) で重い | × |
| **Mip-pyramid Gaussian (Mittring/Kawase)** | Gauss 和 | **O(N) per stage** | **`BloomMethod=Standard`（デフォルト）** |
| Dual Filter Blur (Marius 2015) | Gauss 近似 | down-sample 1tap + up-sample 4tap | 未採用（軽量 Gauss 別系統） |
| **FFT Convolution** | **任意 PSF** | O(N log N) | **`BloomMethod=Convolution`** → [[post_bloom_fft]] |
| Lens Flare（別物） | スプライト合成 | 軽 | LensFlare pass 別途 |

### Mip-pyramid を選ぶ場面

- パフォーマンス優先（コンソール・モバイル含む）
- Gauss でいい look が出せるアートディレクション
- リアルなレンズフレア不要 / 別途 LensFlare assets で補える

### FFT を選ぶ場面 → [[post_bloom_fft]]

- 写真ぽいリアルな bokeh / 回折縞
- カスタムカーネル画像でアーティスト指定の形状
- HDR ハイライトでの自然な diffraction halo

---

## 7. 参考資料

- [S52] Mittring 2007 SIGGRAPH "Finding Next Gen — CryEngine 2"
- Kawase 2003 GDC "Frame Buffer Postprocessing Effects in DOUBLE-S.T.E.A.L (Wreckless)"
- Jimenez 2014 SIGGRAPH "Next Generation Post Processing in Call of Duty: Advanced Warfare"
- 関連: [[post_bloom_fft]] / [[tone_aces]] / [[tone_filmic]]

---

## 8. 相談用フック

- **理解度チェック**:
  - Mip-pyramid が大半径 Gaussian と等価になる根拠 → ガウシアン自己畳み込み (σ² 加算性)
  - Cross Bloom の意味 → カーネル中心強調で疑似 "+" or anamorphic
  - Threshold の物理性 → -1（物理）で全画素 bloom、>0 は古典抽出
- **コード深掘り候補**:
  - `PostProcessBloom.usf:37-67` `BloomSetupCommon` の閾値 / Local Exposure 適用
  - `PostProcessBloomSetup.cpp:225-273` `AddGaussianBloomPasses` の stage ループ
  - `PostProcessWeightedSampleSum.cpp` の 1D Gaussian + Additive 実装
- **未読箇所**:
  - DownsampleChain の構築コスト分析
  - Pre-Exposure 機構と Bloom の数値安定性
  - Mobile Bloom の差分（`PostProcessMobile.cpp` 内）
- **次の派生**:
  - リアル PSF → [[post_bloom_fft]]
  - 出力先 Tonemap → [[tone_aces]] / [[tone_filmic]]
  - LocalExposure 詳細 → 別途解析
