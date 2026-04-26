---
name: SSGI (Screen Space Global Illumination)
description: スクリーンスペース GI の理論と UE 実装（SSR の拡散版 + leak free reproject）
type: project
---

# SSGI — Screen Space Global Illumination

- 上位: [[_algorithm_index]]
- 関連: [[ss_ssr]] / [[ss_ssao]] / [[lumen_final_gather]]
- 採用システム: Lumen 不使用時の動的 GI フォールバック
- 出典:
  - **S42**: Mara et al. 2017 "Screen-Space Indirect Lighting" / SSR の理論を diffuse に拡張した一群の手法
  - UE 実装: `ScreenSpaceRayTracing.cpp` / `SSRT/SSRTDiffuseIndirect.usf`

---

## 1. 何のためのアルゴリズムか

直接照明 + AO だけでは「色付き間接光」が出ない:
- 赤い壁から床へのバウンス
- 緑の草から人物への color bleeding

完全 GI は重いので **画面内のジオメトリから 1〜数バウンス** をスクリーンスペースで近似。

### 素朴な手法の問題

- **VPL (Virtual Point Light)**: 動的更新が重い
- **Lightmap**: 静的のみ
- **Voxel Cone Tracing**: メモリ + 速度コスト
- **RT GI**: 高 GPU 要求

### SSGI の貢献

- **SSR の機構を半球サンプリングで diffuse に流用**
- **前フレーム SceneColor をソース**にすることで多バウンス効果（フィードバック型）
- Lumen の Diffuse Indirect と相互排他（Lumen が SSGI の進化型と言える）

---

## 2. 理論

### 2.1 拡散版 Screen Space Trace

各受信ピクセルから:
1. cosine-weighted half-sphere からサンプル方向 ω_i を N 本（quality=4 で 16 本程度）
2. 各 ω_i を screen-space ray に変換 → HZB march（[[ss_ssr]] §2.3 と同じ）
3. ヒット位置の **前フレーム SceneColor** を読み取り → 入射放射輝度 L_i
4. Lambert: `Indirect += L_i * cos θ_i / N`

### 2.2 前フレーム SceneColor の意味

SSGI が「前フレーム」を読むのは:
- 同フレームでは間接光込みの SceneColor が未完成
- 前フレームを読めば既に N-1 バウンス入った色 → 多バウンス feedback で漸近的に収束

`r.SSGI.LeakFreeReprojection=1`: 前フレーム reproject を leak-free モードで行う（depth が大きく違う場所からの色洩れ抑制）。コスト上昇との trade-off。

### 2.3 Uncertain Ray Rejection

screen-space は「裏側にあるかも」「画面外かも」を判定不能 → uncertain:

```
r.SSGI.RejectUncertainRays=1
```

レイが画面端で外に出た / HZB を通り抜けた場合 = uncertain → そのレイは破棄して N を減らす（バイアスはあるが artifact 抑制）。

### 2.4 Terminate Certain Ray

```
r.SSGI.TerminateCertainRay=1
```

確実に何もヒットしないと分かったレイは fallback（cubemap 等）を呼ばずに 0 で終了 → 不要計算節約。

### 2.5 Sky Distance

```
r.SSGI.SkyDistance=10000000  // KM
```

レイが空に向かって脱出 → 空 cubemap を「この距離にあるとして」評価。物理的には無限遠。

### 2.6 Tile Classification

[[ss_ssr]] と同じ `SSRTTileClassification.usf` を共有。SSGI は基本的に diffuse 全画面なので tile 分類はワーク分散用途寄り。

### 2.7 Temporal + Denoise

SSGI は spp=4〜16 でも noisy。TSR / TAA と独立した GI 専用 temporal を経由（または Lumen と同じ Reservoir 系を後段で）。

### 2.8 Cone Tracing 切替

`SSRTRayCast.ush:18`:
```hlsl
#define SSGI_TRACE_CONE 0
```

過去には cone trace バリアントもあったが現在は ray trace 主流（define が 0）。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `ScreenSpaceRayTracing.cpp` | SSGI / SSR 共通 セットアップ、CVar |
| `SSRT/SSRTDiffuseIndirect.usf` | SSGI メインシェーダ |
| `SSRT/SSRTRayCast.ush` | レイキャスト共通 |
| `SSRT/SSRTPrevFrameReduction.usf` | 前フレーム SceneColor の mip pyramid 生成 |

### 3.2 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.SSGI.Quality` | 4 | サンプル数（1〜4） |
| `r.SSGI.LeakFreeReprojection` | 1 | leak-free 再投影（高品質） |
| `r.SSGI.MinimumLuminance` | 0.5 | 検出閾値（極暗ピクセルは無視） |
| `r.SSGI.RejectUncertainRays` | 1 | 不確実レイ破棄 |
| `r.SSGI.TerminateCertainRay` | 1 | 確実 miss で fallback 抑止 |
| `r.SSGI.SkyDistance` | 1e7 | 空想定距離 (KM) |

### 3.3 PostProcessVolume

- `DynamicGlobalIlluminationMethod = ScreenSpace` を設定すると有効化（`SupportScreenSpaceDiffuseIndirect`）
- Forward Shading は非対応（`IsForwardShadingEnabled` で reject）
- ViewState（履歴）が必要 → エディタ Preview の一部状況で無効

### 3.4 SSR との共有

`ScreenSpaceRayTracing.cpp` 内に SSR / SSGI 両 CVar が並んでいる通り、HZB march 部分のコードを共有。違いは「半球 cosine-weighted vs 鏡面 GGX」「SceneColor source」。

---

## 4. 近似・省略の差分

| 項目 | 理想 GI | UE SSGI | 影響 |
|------|--------|--------|------|
| バウンス数 | ∞ | feedback 型で漸近 | 数フレーム遅延 |
| Off-screen 寄与 | 完全 | 取得不可 | ライトが画面外なら GI 消失 |
| サンプル数 | 数百 | 4〜16 | denoise 必須 |
| 多波長 | 任意 | RGB のみ | spectral 効果なし |
| 透過 | 任意 | opaque のみ | ガラス透過 GI 不可 |

---

## 5. パラメータと CVar

§3.2 にまとめ済み。Lumen に切り替える場合は `DynamicGlobalIlluminationMethod=Lumen`。

---

## 6. 代替手法との比較

| 手法 | 動的 | 多バウンス | UE 採用 |
|------|------|----------|--------|
| Lightmap GI | 不可 | 任意 | 静的 |
| LPV (Light Propagation Volume) | 動的 | 1〜2 | UE4 廃止 |
| **SSGI (Mara 2017 系)** | **動的** | **feedback** | **Lumen 不使用時** |
| **Lumen GI** | **動的** | **多バウンス + Surface Cache** | **UE5 標準** → [[lumen_final_gather]] |
| HW RT GI | 動的 | RT サンプル数 | High-end |
| RTX GI / DDGI | 動的 | プローブベース | プラグイン |

### Lumen との関係

Lumen の Final Gather も内部で SS Trace を最初に試し、外れたら SDF / HW RT に降りる。**SSGI は Lumen の縮約版** と理解すると見通しが良い。Lumen 不使用 / モバイル / 軽量プロジェクト用フォールバックの位置付け。

---

## 7. 参考資料

- [S42] Mara et al. 2017 / Stachowiak 2015 系列
- McGuire & Mara 2014 "Efficient GPU Screen-Space Ray Tracing"
- Karis 2021 [[lumen_final_gather]] への接続
- 関連: [[ss_ssr]] / [[ss_ssao]] / [[lumen_final_gather]]

---

## 8. 相談用フック

- **理解度チェック**:
  - SSGI が前フレーム SceneColor を読む意味 → 多バウンス feedback
  - Uncertain ray の処理 → 棄却することで artifact を抑える代わりにバイアス
  - LeakFreeReprojection の必要性 → depth 段差で色が漏れるのを防ぐ
- **コード深掘り候補**:
  - `SSRTDiffuseIndirect.usf` の半球サンプル分布
  - `SSRTPrevFrameReduction.usf` の mip 生成
  - `SSRTRayCast.ush` の uncertain 判定
- **未読箇所**:
  - SSGI と Lumen の切替境界（`DynamicGlobalIlluminationMethod` 評価）
  - LeakFreeReprojection の具体実装
  - SSGI 用 denoise（履歴ブレンド）の構成
- **次の派生**:
  - Lumen 統合 GI → [[lumen_final_gather]]
  - 反射版 → [[ss_ssr]]
  - AO フォールバック → [[ss_ssao]]
