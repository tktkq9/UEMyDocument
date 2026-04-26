---
name: PCF / PCSS Shadow Filtering
description: Percentage Closer Filtering と Percentage Closer Soft Shadows の理論と UE 実装
type: project
---

# PCF / PCSS（Shadow Filtering）

- 上位: [[_algorithm_index]]
- 関連: [[vsm_overview]] / [[vsm_page_streaming]]
- 採用システム: VSM / 旧シャドウマップ系のソフトシャドウ
- 出典 ID: **S22**（[[_source_index]]）— Reeves et al. 1987 SIGGRAPH (PCF 原典)
- 関連: Fernando 2005 "Percentage-Closer Soft Shadows" (PCSS 原典)
- UE 実装: `ShadowProjectionCommon.ush` / `VirtualShadowMapProjection.usf`

---

## 1. 何のためのアルゴリズムか

シャドウマップ単純比較は **完全ハード**（ピクセル単位の二値）で:
- ジャギー（aliasing）が目立つ
- 接触陰がにじまず非物理的（半影 = penumbra がない）

### 素朴な手法の問題

- **単純 depth compare**: 1 サンプル比較 → ジャギー
- **マップを blur**: 深度 blur は遠近不正（シルエット周辺がにじむ）
- **VSM (Variance SM) / ESM**: light leaking 問題

### PCF / PCSS の貢献

- **PCF**: 周辺 N サンプルで depth compare → 平均化（"何 % が closer"）→ ハード境界を抗エイリアシング
- **PCSS**: ブロッカーまでの距離からカーネル半径を動的決定 → **物理的に妥当な半影**（接触陰は鋭く、遠い陰は柔らかく）

---

## 2. 理論

### 2.1 PCF（Percentage Closer Filtering, Reeves 1987）

```
shadow = 0
for each (offset_x, offset_y) in kernel:
    z_sample = ShadowMap.Sample(uv + offset)
    if z_sample < z_receiver:
        shadow += 1
shadow /= NumSamples
```

- カーネルは正方 / Poisson disk / Vogel disk
- **比較を先に**してから平均（depth blur ではない！）
- HW: `SamplerComparisonState` + `SampleCmp` で 4 サンプル PCF 1 命令

### 2.2 PCSS（Percentage Closer Soft Shadows, Fernando 2005）

```
[1] Blocker Search:
    軽量カーネルでブロッカー（z < z_receiver）の平均深度 zBlocker を取得
[2] Penumbra Estimation:
    wPenumbra = (z_receiver - zBlocker) / zBlocker × lightSize
[3] PCF with adaptive kernel:
    カーネル半径 = wPenumbra × scale で PCF 実行
```

物理モデル: 光源が点でなく面積を持つと、ブロッカーが遠いほど penumbra が広がる（影が柔らかくなる）。`zBlocker` が `z_receiver` に近い（接触）= 鋭い陰、遠い = 柔らかい陰。

### 2.3 Poisson / Vogel Disk サンプル

均一格子サンプルは banding が出るので、低差異シーケンス（Poisson disk / Vogel disk）を用いる:

```hlsl
// Vogel disk (golden angle)
float2 SampleVogelDisk(int i, int N, float phi)
{
    float r = sqrt((i + 0.5) / N);
    float theta = i * 2.4f + phi;  // 2.4 ≈ golden angle in rad
    return r * float2(cos(theta), sin(theta));
}
```

回転角 `phi` をピクセル毎ランダム化すれば spatial 多様性 → TAA で時系列に解消される。

### 2.4 Receiver Plane Depth Bias

斜面で self-shadowing acne を避けるため、サンプルオフセットに応じた深度補正:

```
zCorrected = zReceiver + dot(ddxy(zReceiver), offset)
```

`ShadowProjectionCommon.ush` の `RPDB`（Receiver Plane Depth Bias）で実装。

### 2.5 Cone Tracing 風アプローチ（VSM/SMRT）

UE5 VSM では PCF よりさらに **SMRT (Stochastic Many-Light Ray Tracing)** 風に複数サンプルでブロッカー探索 → ノイズは TSR / TAA で解消。完全 PCSS ではないが、似た penumbra 制御を実現。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `ShadowProjectionCommon.ush` | PCF カーネル、receiver plane bias |
| `VirtualShadowMapProjection.usf` | VSM レシーバ側サンプリング |
| `VirtualShadowMapProjection.cpp` | VSM 投影 pass オーケストレーション |
| `ShadowFilteringCommon.ush` | サンプリング関数群 |

### 3.2 主要 CVar（旧シャドウマップ系 / VSM 共通）

| CVar | 効果 |
|------|------|
| `r.Shadow.MaxResolution` | シャドウマップ最大解像度 |
| `r.Shadow.RadiusThreshold` | 微小シャドウのカット |
| `r.Shadow.Virtual.SMRT.RayCountDirectional` | VSM SMRT サンプル数 (Directional) |
| `r.Shadow.Virtual.SMRT.RayCountLocal` | VSM SMRT サンプル数 (Local) |
| `r.Shadow.Virtual.SMRT.SamplesPerRayDirectional` | レイあたりサンプル |
| `r.Shadow.Virtual.SMRT.MaxRayAngleFromLight` | ライト面積近似角度 |
| `r.Shadow.Virtual.SMRT.RayLengthScaleDirectional` | レイ長スケール |
| `r.Shadow.FilterMethod` | フィルタ方式（PCF など） |

（CVar は UE バージョンで増減するので参考値。`r.Shadow.Virtual.SMRT.*` ファミリは VSM 専用）

### 3.3 受信側サンプリング（VSM）

VSM では受信ピクセルが:
1. ライト座標系のシャドウ座標を計算
2. 仮想ページ ID を引く
3. ページテーブルから物理ページを取得
4. SMRT 風サンプルカーネルで深度比較

mip 階層から「適切な解像度」のページを選ぶことで、ピクセル footprint に応じたフィルタ品質を担保。

---

## 4. 近似・省略の差分

| 項目 | 理想 | UE 実装 | 影響 |
|------|------|--------|------|
| 半影モデル | エリアライト面積積分 | PCSS/SMRT 近似 | ペナンブラ形状の物理性は十分実用 |
| サンプル数 | 無限 | 8〜16 サンプル + TAA | TAA 無効時にノイズ |
| Banding | 0 | Poisson/Vogel disk | 低周波 banding 残留 |
| Receiver Bias | 完全 | RPDB（一次近似） | 非常に斜め面で acne |
| Light 形状 | 任意 | 半径円盤近似 | 矩形ライトは LTC（[[area_light_ltc]]）と組合せ |

---

## 5. パラメータと CVar

§3.2 にまとめ済み。VSM 利用時は `r.Shadow.Virtual.SMRT.*` を主に調整。

---

## 6. 代替手法との比較

| 手法 | 半影制御 | leaking | UE 採用 |
|------|--------|--------|--------|
| Hard Shadow | なし | なし | デバッグ |
| **PCF (Reeves 1987)** | **固定半径** | **なし** | **基本フィルタ** |
| **PCSS (Fernando 2005)** | **動的半径** | **なし** | **論文** |
| VSM (Variance SM, Donnelly 2006) | depth blur | あり | 非採用 |
| ESM (Exponential SM) | exp 圧縮 | あり | 非採用 |
| MSM (Moment SM) | モーメント | 軽微 | 非採用 |
| **SMRT (UE5 VSM)** | **Stochastic ray** | **なし** | **VSM 標準** |

### PCSS vs SMRT

- PCSS: 2 段（blocker search → adaptive PCF）
- SMRT: 1 段でレイトレース風サンプル → ブロッカー深度を平均し penumbra を推定
- SMRT は VSM の mip 階層と相性が良く、TSR でノイズが消えやすい

---

## 7. 参考資料

- [S22] Reeves et al. 1987 "Rendering Antialiased Shadows with Depth Maps"
- Fernando 2005 "Percentage-Closer Soft Shadows" (NVIDIA)
- Donnelly & Lauritzen 2006 "Variance Shadow Maps"（比較対象）
- Karis 2021 SIGGRAPH §5 — VSM SMRT の議論
- 関連: [[vsm_overview]] / [[area_light_ltc]]（未着手）

---

## 8. 相談用フック

- **理解度チェック**:
  - PCF が depth blur と違う点 → "比較してから平均" であって "平均してから比較" ではない
  - PCSS の半径計算式 → `(z - zBlocker) / zBlocker × lightSize`
  - SMRT が PCSS より好まれる理由 → ノイズが TSR と協調、サンプル分布の柔軟性
- **コード深掘り候補**:
  - `ShadowProjectionCommon.ush` の RPDB 実装
  - `VirtualShadowMapProjection.usf` の SMRT カーネル
  - `ShadowFilteringCommon.ush` の Vogel disk 関数
- **未読箇所**:
  - SMRT のレイ長 / サンプル分布の詳細式
  - Hair / Cloth 等の anisotropic shadow 統合
  - Local Light の球状ライト面積モデル
- **次の派生**:
  - 矩形エリアライト → [[area_light_ltc]]（未着手）
  - VSM 概要 → [[vsm_overview]]
  - Hair shadow → 未着手
