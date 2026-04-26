---
name: FFT Bloom (Convolution / Lens Bloom)
description: 周波数領域畳み込みによる物理 Bloom の理論と UE 実装（GPUFastFourierTransform + spectral kernel cache）
type: project
---

# FFT Bloom — Lens Convolution

- 上位: [[_algorithm_index]]
- 関連: [[post_bloom]] / [[tone_aces]]
- 採用システム: 写真ぽい lens bloom（任意 PSF 画像をユーザー供給）。標準 Mip-pyramid と相互排他
- 出典:
  - **S53**: Cooley & Tukey 1965 "An Algorithm for the Machine Calculation of Complex Fourier Series" (FFT 基礎)
  - 関連: Convolution Theorem `f * g  ⇔  F · G`
  - UE 実装: `PostProcessFFTBloom.cpp` / `Bloom/*.usf` (11 ファイル) / `GPUFastFourierTransform.cpp/.usf`

---

## 1. 何のためのアルゴリズムか

[[post_bloom]] の Mip-pyramid Gaussian は速いが「Gaussian しか作れない」。FFT Bloom は **任意の PSF（点像分布関数）** を画像として与え、それでシーンを畳み込む:

- カメラレンズの絞り回折（六角・八角絞り）
- アナモルフィックレンズの横長レイ
- スターフィルタ系（クロス・スノーフレーク）
- アーティスト手書きカスタム

物理的には光学系の PSF をそのまま再現でき、HDR ハイライトに「カメラが映したらこう見える」という説得力を出す。

### Mip-pyramid の限界

- カーネル形状が Gauss 固定 → 絞り羽根の数や形は出せない
- 多段加算で複雑形状を近似はできるが芸術的調整が困難
- 写真比較で違和感（絞り回折縞は Gauss では出ない）

---

## 2. 理論

### 2.1 畳み込み定理

時間/空間領域の畳み込み = 周波数領域の点積:

```
(f * g)(x) = F⁻¹{ F{f} · F{g} }(x)
```

`f`: SceneColor (画像)、`g`: PSF カーネル。

直接畳み込み: O(N · K)（K=カーネル画素数、デカいほど重い）  
FFT 経由: O(N log N)（K に非依存）

カーネルが「画像全体」級に大きい場合 FFT が圧勝。Bloom はまさにこのケース（数百 px 半径）。

### 2.2 FFT Bloom パイプライン

```
1. SceneColor → ForwardFFT(2D) → SpectralImage
2. KernelImage → ForwardFFT(2D) → SpectralKernel  ← cache 可能
3. SpectralResult = SpectralImage · SpectralKernel  (complex multiply)
4. SpectralResult → InverseFFT(2D) → BloomedImage
5. SceneColor + Intensity * BloomedImage → 出力
```

`r.Bloom.CacheKernel=1`: kernel が変化しなければ Step 2 を初回のみ実行 → 毎フレーム省略。

### 2.3 GPUFastFourierTransform

UE は専用ライブラリ `GPUFastFourierTransform.cpp/.usf` を持ち、**Stockham FFT** を Compute Shader で実装:
- 2D は「行 FFT → 列 FFT」分解
- 行/列 各 1024 〜 2048 px までサポート（共有メモリと register pressure 制約）
- complex 値は R32G32_FLOAT or float2

`r.Bloom.ScreenPercentage` (`PostProcessFFTBloom.cpp:17-20`) で FFT 解像度を絞れる:
- 50%: 1/4 ピクセル → 4 倍速、品質低下
- 100%: フル
- >100% は不要（カーネル広がりがあるので普通は <=100）

### 2.4 Kernel Pre-Process（Bloom/*.usf 群）

ユーザー画像をそのまま FFT に渡せず、**前処理 11 段** が必要:

| Pass | シェーダ | 役割 |
|------|---------|------|
| FindKernelCenter | `BloomFindKernelCenter.usf` | カーネル中央輝点座標を検出 |
| SurveyMaxScatterDispersion | `BloomSurveyMaxScatterDispersion.usf` | カーネル外周の最大エネルギー測定 |
| SurveyKernelCenterEnergy | `BloomSurveyKernelCenterEnergy.usf` | 中央スパイクのエネルギー測定 |
| ReduceKernelSurvey | `BloomReduceKernelSurvey.usf` | survey buffer の reduce（max/sum） |
| SumScatterDispersionEnergy | `BloomSumScatterDispersionEnergy.usf` | 散乱成分エネルギー合計 |
| PackKernelConstants | `BloomPackKernelConstants.usf` | 前段の数値を `KernelConstantsBuffer` にパック |
| ClampKernel | `BloomClampKernel.usf` | カーネル中央スパイクを clamp（DC 残し） |
| DownsampleKernel | `BloomDownsampleKernel.usf` | カーネル縮小（FFT 解像度に合わせる） |
| ResizeKernel | `BloomResizeKernel.usf` | 最終 FFT サイズに padding/centering |
| FinalizeApplyConstants | `BloomFinalizeApplyConstants.usf` | spectral 乗算後の正規化定数を適用 |

#### Center Spike Clamp の意義

カーネル中央のスパイク（PSF の中心輝点）をそのまま乗じると **元 SceneColor が二重に出る** → bloom 加算で破綻。中央スパイクのエネルギーを別計算で吸収し、`ScatterDispersionIntensity` (`PostProcessFFTBloom.cpp:207`) で **散乱成分のみ** を正規化加算。これで「シーン + 散乱光」になり物理整合。

### 2.5 Spectral Kernel Cache

`r.Bloom.CacheKernel=1` (`PostProcessFFTBloom.cpp:22-25`):

- カーネル画像が変化しない限り、SpectralKernel を `FSceneViewState` 等に保持
- Pre-Process 11 段を初回のみ実行
- 毎フレーム必要なのは **Forward FFT(Scene) + Multiply + Inverse FFT(Result)** のみ

カーネル変更タイミング:
- `BloomConvolutionTexture` 差し替え
- カーネル size scale 変更
- 解像度変更
- Stage 切替

### 2.6 KernelSupportScale と境界

`FBloomResizeKernelCS` (`PostProcessFFTBloom.cpp:178-198`) の `KernelSupportScale`:
- カーネルが画面端より大きいと wrap-around (FFT は周期境界) → ハロー漏れ
- カーネルを zero-padding して画面サイズより大きい FFT バッファに置く
- `KernelSupportScale=1.0` で「カーネル直径 = 画面長辺」、>1 で余裕持たせる

### 2.7 Async Compute

`r.Bloom.AsyncCompute=1` (`PostProcessFFTBloom.cpp:27-30`):
- FFT は ALU 重・dependency 長い → ラスタライズと並列に async queue で走らせ
- 描画パスと overlap してフレーム時間短縮

### 2.8 Warn Kernel Resolution

`r.Bloom.WarnKernelResolution`:
- ユーザー カーネル画像が必要以上に高解像（FFT 解像度より大きい）→ 無駄
- `2`: 全プラットフォーム警告、`1`: コンソールのみ、`0`: 無効
- 4K カーネル → 1080p FFT に縮小されているのに気付くため

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `PostProcessFFTBloom.cpp/.h` | FFT Bloom pass セットアップ、CVar、Pre-Process 11 段 dispatch |
| `Bloom/BloomFindKernelCenter.usf` | カーネル中央輝点検出 |
| `Bloom/BloomSurveyMaxScatterDispersion.usf` | 散乱最大値 |
| `Bloom/BloomSurveyKernelCenterEnergy.usf` | 中央エネルギー |
| `Bloom/BloomReduceKernelSurvey.usf` | survey reduce |
| `Bloom/BloomSumScatterDispersionEnergy.usf` | 散乱合計 |
| `Bloom/BloomPackKernelConstants.usf` | 定数 pack |
| `Bloom/BloomClampKernel.usf` | 中央 spike clamp |
| `Bloom/BloomDownsampleKernel.usf` | カーネル縮小 |
| `Bloom/BloomResizeKernel.usf` | FFT サイズ調整 |
| `Bloom/BloomFinalizeApplyConstants.usf` | 正規化定数適用 |
| `Bloom/BloomCommon.ush` | `FBloomKernelInfo` 構造体 |
| `GPUFastFourierTransform.cpp/.usf` | 汎用 FFT/IFFT （Bloom 以外でも使用） |

### 3.2 FBloomKernelInfo（`BloomCommon.ush:14-30`）

```c
struct FBloomKernelInfo
{
    float4 CenterEnergy;             // 中央スパイクエネルギー
    float4 ScatterDispersionEnergy;  // 散乱（halo）エネルギー
    float4 MaxScatterDispersionEnergy; // 散乱の per-pixel 上限
    uint2  CenterPixelCoord;          // 中央座標
    uint2  _Padding;                  // Vulkan alignment
};
```

これがカーネル前処理結果を集約 → 本処理 dispatch に渡される。

### 3.3 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.Bloom.ScreenPercentage` | 100 | FFT 軸解像度（50〜100） |
| `r.Bloom.CacheKernel` | 1 | spectral kernel をキャッシュ |
| `r.Bloom.AsyncCompute` | 1 | async compute で並列実行 |
| `r.Bloom.WarnKernelResolution` | 1 | カーネル解像度警告（0/1/2） |

### 3.4 PostProcess Settings

| 名前 | 用途 |
|------|------|
| `BloomMethod` | `Convolution` で FFT 経路有効化 |
| `BloomConvolutionTexture` | カーネル画像（Texture2D） |
| `BloomConvolutionScatterDispersion` | 散乱強度（[[post_bloom]] §3.3 の `ScatterDispersionIntensity` に流入） |
| `BloomConvolutionSize` | カーネルの相対サイズ |
| `BloomConvolutionCenterUV` | カーネル中央 UV（auto detect 結果上書き） |
| `BloomConvolutionPreFilter` | スーパーブライト抽出閾値（min/max/mult） |
| `BloomConvolutionBufferScale` | バッファ拡張倍率（halo 余裕） |

### 3.5 Platform 制限

`DoesPlatformSupportFFTBloom(Platform)` (`PostProcessFFTBloom.cpp:40-43`) が `FDataDrivenShaderPlatformInfo::GetSupportsFFTBloom()` を参照:
- D3D12 / Vulkan / Metal の SM5+ で有効
- モバイル / 旧世代コンソール / ES3.1 等で無効
- 非対応プラットフォームでは自動的に Mip-pyramid にフォールバック

### 3.6 動作フロー（フレーム毎）

```
[初回 or kernel 変更時]
    KernelImage Load
    FindKernelCenter → SurveyMax/Center → ReduceSurvey
    SumScatter → PackConstants
    ClampKernel → DownsampleKernel → ResizeKernel
    ForwardFFT(Kernel) → SpectralKernel cache

[毎フレーム]
    SceneDownsample → BloomSetup
    ForwardFFT(SceneColor)  ← async
    Multiply by SpectralKernel
    InverseFFT
    FinalizeApplyConstants  → SceneColor 加算
```

---

## 4. 近似・省略の差分

| 項目 | 理想 | UE FFT Bloom | 影響 |
|------|------|------------|------|
| カーネル形状 | 任意の連続 PSF | 入力テクスチャ離散 | 解像度依存のサンプリング誤差 |
| 計算量 | O(N log N) | 同じ + 11 段前処理 | 初回コスト高、cache で償却 |
| 境界 | 真の畳み込み | FFT 周期境界 → padding 必須 | KernelSupportScale 不足で wrap |
| 色域 | wavelength 別可 | RGB 各々 FFT | 単純色収差のみ、 真の chromatic aberration なし |
| 数値精度 | float64 | float32（spectral も） | 巨大 FFT で誤差累積 |

---

## 5. パラメータと CVar

§3.3 / 3.4 にまとめ済み。Cinematic 用途は `r.Bloom.ScreenPercentage=100` + `r.Bloom.CacheKernel=1` + `BloomConvolutionBufferScale` を 1.5〜2 推奨。

---

## 6. 代替手法との比較

| 手法 | カーネル形状 | 速度 | UE 採用 |
|------|------------|------|--------|
| Single Direct Convolution | 任意 | O(N·K)、巨大 K で破綻 | × |
| **Mip-pyramid Gaussian** | **Gauss のみ** | O(N) | **`BloomMethod=Standard`** → [[post_bloom]] |
| **FFT Convolution** | **任意 PSF** | **O(N log N)** | **`BloomMethod=Convolution`** |
| Sparse Bloom (per-bright-pixel sprite) | 1 PSF stamp | bright pixel 数次第 | × （UE 未採用） |
| Dual-Filter Blur | Gauss 近似 | 軽量 | 未採用 |

### Mip-pyramid と FFT のクロスオーバー

| 観点 | Mip-pyramid | FFT |
|------|-------------|-----|
| カーネル形状 | Gauss のみ | 任意画像 |
| 初回コスト | なし | Pre-Process 11 段 |
| 毎フレームコスト | low | medium（FFT 2 回 + 乗算） |
| メモリ | 低 | 高（spectral buffer） |
| アート寄与 | tint × 6 | 任意 PSF テクスチャ |
| プラットフォーム | 全対応 | SM5+ のみ |

シネマ / フォトリアル → FFT。ゲーム実機 / モバイル → Mip-pyramid。

---

## 7. 参考資料

- [S53] Cooley & Tukey 1965 "An Algorithm for the Machine Calculation of Complex Fourier Series"
- Goodman 2005 "Introduction to Fourier Optics"（光学 PSF と畳み込み）
- Kraus & Strengert 2007 "Depth-of-Field Rendering by Pyramidal Image Processing"（FFT 系議論）
- Stockham 1966 "High-speed convolution and correlation"
- 関連: [[post_bloom]] / [[tone_aces]]

---

## 8. 相談用フック

- **理解度チェック**:
  - 畳み込み定理 → 空間 conv = 周波数 product
  - Center Spike Clamp の意義 → 中央スパイクを残すと SceneColor が二重に出るので散乱のみ加算
  - Spectral Kernel Cache の効果 → Pre-Process 11 段を 0 円に
- **コード深掘り候補**:
  - `PostProcessFFTBloom.cpp:178-198` `BloomResizeKernelCS` の padding ロジック
  - `Bloom/BloomFindKernelCenter.usf` の重心検出
  - `GPUFastFourierTransform.usf` の Stockham FFT 実装
- **未読箇所**:
  - 1024 vs 2048 FFT の閾値（解像度自動切替）
  - Async Compute スケジューリングと SceneColor 依存
  - HDR Bright clamp との順序
- **次の派生**:
  - 軽量 Bloom → [[post_bloom]]
  - Tonemap 直前との合流 → [[tone_aces]]
  - Lens Flare 個別パス → 別途解析
