---
name: SSAO (Mittring HBAO 系)
description: スクリーンスペース AO の理論と UE 実装（Crytek/Mittring 系 horizon based）
type: project
---

# SSAO — Mittring HBAO 系

- 上位: [[_algorithm_index]]
- 関連: [[ss_ssr]] / [[ss_ssgi]] / [[lumen_final_gather]]
- 採用システム: 旧来 / モバイル AO（Lumen 不使用時のフォールバック）
- 出典 ID: **S40**（[[_source_index]]）— Mittring 2007 SIGGRAPH "CryEngine 2"（SSAO 原典）
- 影響元: Bavoil & Sainz 2008 "Image-Space Horizon-Based AO" (HBAO), Bunnell 2005
- UE 実装: `PostProcessAmbientOcclusion.cpp` / `.usf` / `.ush`

---

## 1. 何のためのアルゴリズムか

直接照明だけでは「凹み」「接触面の暗さ」が再現できず CG ぽい。物理的には Ambient Occlusion 項:

```
AO(p) = (1/π) ∫_Ω V(p, ω) cos θ dω
```

を deferred で安価に近似する。

### 素朴な手法の問題

- **Per-pixel ray cast**: GPU で重すぎ
- **Pre-baked AO**: 動的物体非対応
- **Voxel AO**: メモリ重

### Mittring/HBAO の貢献

- 深度バッファ + 法線のみで「ピクセル周辺がどれだけ閉じているか」を **半球サンプリング**で推定
- HBAO: 各方向の **horizon angle** を depth から求めて積分式に近似
- Half-res / Quarter-res パスで負荷削減 + bilateral upsample

---

## 2. 理論

### 2.1 SSAO 基本式（Mittring 2007）

```
SSAO(p) = 1 - (1/N) Σ_i V(p_i)
```

`p_i` = 半球内ランダムサンプル（ノーマル方向）。`V(p_i)` は深度バッファで `z_sample < z_geometry` なら 1。

### 2.2 HBAO（Horizon-Based）

各方向 φ に対して horizon angle h(φ) を depth march で探索:

```
AO = 1 - (1/2π) ∫₀^{2π} (sin h(φ) - sin n(φ)) dφ
```

`n(φ)` は表面の傾斜角。離散化は方向 8 / 各方向ステップ 4 程度。

### 2.3 UE のハイブリッド（Setup → AO → Blur → Composite）

UE 実装は HBAO 型ではなく **Horizon search を簡略化した Mittring 系**:

1. **Setup**: depth + normal を半解像度に downsample
2. **AO compute**: 半球内 N サンプル、HZB から深度を取り visibility 評価
3. **Blur**: Bilateral filter（深度差で重みづけ、edge 保持）
4. **Composite**: Lit に乗算

サンプルパターンはランダム Normal Texture と Vogel disk（`RandomNormalTexture`）。

### 2.4 HZB との連動

`PostProcessAmbientOcclusionCommon.ush:31-38` で `HZB.ush` を include し:

```hlsl
float GetHZBDepth(float2 ScreenPos, float MipLevel)
{
    float2 HZBUV = ApplyScreenTransform(ScreenPos, ScreenPosToHZBUVScaleBias);
    float HZBDepth = Texture2DSampleLevel(HZBTexture, HZBSampler, HZBUV, MipLevel).r;
    return ConvertFromDeviceZ(HZBDepth);
}
```

サンプル距離が遠いほど HZB の高 mip を引く → cache friendly + 偽陰影抑制。

### 2.5 Bilateral Upsample（ハーフ解像度→フル解像度）

```
weight_i = depthSimilarity(z_i, z_full) × normalSimilarity(n_i, n_full)
result = Σ weight_i × ao_i / Σ weight_i
```

`ComputeDepthSimilarity` は `1 - abs(zA - zB) × scale`（`PostProcessAmbientOcclusionCommon.ush:60-63`）。エッジ越えで重み 0 → 漏れ抑制。

### 2.6 Temporal Smoothing

ノイズが多いので前フレーム履歴と blend（TAA / TSR と独立した AO 専用 temporal）。`r.AmbientOcclusion.Method` などで挙動切替。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `PostProcessAmbientOcclusion.cpp` | パスセットアップ、CVar |
| `PostProcessAmbientOcclusion.usf` | Setup / AO / Blur shader |
| `PostProcessAmbientOcclusionCommon.ush` | 共通関数（ReconstructCSPos, GetHZBDepth, Bilateral） |
| `PostProcessAmbientOcclusionMobile.cpp/.usf` | モバイル版（軽量化） |

### 3.2 主要 CVar / Settings

| 名前 | 用途 |
|------|------|
| `r.AmbientOcclusionLevels` | パス段数（通常 3） |
| `r.AmbientOcclusionRadiusScale` | 半径スケール |
| `r.AmbientOcclusionStaticFraction` | 静的 vs 動的の混合 |
| `r.AmbientOcclusion.Method` | 0=既定, 1=GTAO 等 |
| `r.AmbientOcclusion.Compute` | 0=PS, 1=CS |
| `r.AmbientOcclusion.AsyncCompute` | async compute |
| `r.HZBOcclusion` | HZB 構築連動 |

### 3.3 ScreenSpaceAOParams 配列（`*Common.ush:8-13`）

```c
// [0]: AO Power / Bias / 1/Distance / Intensity
// [1]: ViewportUVToRandomUV / Radius / Ratio
// [2]: ScaleFactor / InvThreshold / WS-or-VS / MipBlend
// [3]: TemporalAARandomOffset / StaticFraction / InvTanHalfFov
// [4]: FadeDistance Multiplier / Additive / HzbStepMipLevelFactor / unused
float4 ScreenSpaceAOParams[5];
```

PostProcessVolume → `FAmbientOcclusionSettings` → このパラメータ列に圧縮される。

---

## 4. 近似・省略の差分

| 項目 | 理想 | UE SSAO | 影響 |
|------|------|--------|------|
| サンプル数 | 無限 | 4〜16 + temporal | ノイズ残留 |
| Visibility | 真の遮蔽 | depth バッファ近似 | 物体裏側が見えず誤陰影 |
| 距離 | 全空間 | 半径制限 | 遠距離 AO 無し |
| Normal | 法線バッファ | reconstruct 可 | 平坦面で不安定 |
| 動的 vs 静的 | 統一 | StaticFraction 補正 | ライトベイクとの齟齬 |

---

## 5. パラメータと CVar

PostProcessVolume 設定が主、CVar は scalability 用。

---

## 6. 代替手法との比較

| 手法 | 種別 | UE 採用 |
|------|------|--------|
| **Crytek SSAO (Mittring 2007)** | image-space, half-res | **UE 標準 SSAO** |
| HBAO (Bavoil 2008) | horizon based | プラグイン的 |
| GTAO (Jimenez 2016) | ground truth, 物理整合 | UE5 一部 path |
| Voxel AO (VXAO) | 体積近似 | NVIDIA gameworks |
| **Lumen Final Gather** | **本物のラジオシティ** | **UE5 標準（GI 兼ねる）** → [[lumen_final_gather]] |
| RTAO | DXR raytrace | UE 5 オプション |

### Lumen 時代の SSAO

Lumen の Final Gather が AO 込みの間接光を生成するため、**Lumen 有効時 SSAO はオフが推奨**。SSAO は Lumen 不使用 / モバイル / Forward 用フォールバック。

---

## 7. 参考資料

- [S40] Mittring 2007 SIGGRAPH "Finding Next Gen"
- Bavoil & Sainz 2008 "Image-Space Horizon-Based AO"
- Jimenez et al. 2016 "Practical Realtime Strategies for Accurate Indirect Occlusion"（GTAO）
- 関連: [[ss_ssr]] / [[ss_ssgi]] / [[lumen_final_gather]]

---

## 8. 相談用フック

- **理解度チェック**:
  - SSAO が深度バッファだけで AO 推定できる仕組み → サンプル位置の z と geo z の比較
  - HZB を使う理由 → 遠距離サンプルで cache 効率と smooth な visibility
  - Bilateral upsample の動機 → ハーフ解像 AO のエッジを保持しながら拡大
- **コード深掘り候補**:
  - `PostProcessAmbientOcclusion.usf` の MainAOPS
  - `*Common.ush:60-71` の depth similarity / smaller delta
  - Mobile 版の simplification
- **未読箇所**:
  - GTAO 統合の現状（UE5 path）
  - Vogel disk vs RandomNormalTexture の比較
  - Substrate との連携
- **次の派生**:
  - SSR → [[ss_ssr]]
  - SSGI → [[ss_ssgi]]
  - Lumen 最終集約 → [[lumen_final_gather]]
