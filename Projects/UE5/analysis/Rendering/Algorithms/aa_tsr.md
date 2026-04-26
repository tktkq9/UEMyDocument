---
name: TSR (Temporal Super Resolution, Epic SIGGRAPH 2022)
description: TSR の多段アーキテクチャと UE5 実装（Velocity / Reject / Update / Resolve）
type: project
---

# TSR — Temporal Super Resolution（Epic SIGGRAPH 2022）

- 上位: [[_algorithm_index]]
- 関連: [[aa_taa]] / [[nanite_visibility_buffer]]
- 採用システム: UE5 標準 AA / Upscaler（プロジェクト設定 `r.AntiAliasingMethod=4`）
- 出典:
  - **S31**: Epic 2022 SIGGRAPH "Lyra & TSR" / Epic Online Learning "TSR Deep Dive"
  - 影響元: TAAU（UE4.23）, Karis 2014 [[aa_taa]], NN-based upscaler 系（DLSS/FSR）
  - UE 実装: `PostProcess/TemporalSuperResolution.cpp` / `TemporalSuperResolution/*.usf`

---

## 1. 何のためのアルゴリズムか

TAA / TAAU には限界:
- **ヒストリ rejection の粗さ** → ghost / smear が頻発
- **upsample 倍率を大きくすると aliasing 残留**（4K from 1080p 等）
- **シェーディング変化（ライト点滅 / 反射移動）に反応せず ghost**

### TSR の貢献

- **Shading Rejection**: 履歴と現フレームのシェーディング差を per-pixel で検出 → 不一致なら履歴を破棄
- **Flickering Heuristic**: モアレ / 高密度ジオメトリで luma が定常振動する区画を検出 → 範囲内ゴーストで抑制
- **Convolution Network**: TSR Reject Pass で 32×32 タイルに小型 CNN 風カーネルを適用しシェーディング差を分類
- **大幅 upsample（最大 100% → 33%）でも安定**
- **Reprojection Field**: ピクセル単位の最小投影距離で disocclusion を検出
- **Thin Geometry Detection**: ワイヤ / 髪などの 1 ピクセル以下構造専用パス

---

## 2. 理論（パス構成）

TSR は単一 shader ではなく **多段 compute pipeline**:

| パス | usf | 役割 |
|------|------|------|
| **ClearPrevTextures** | `TSRClearPrevTextures.usf` | 履歴初期化 |
| **MeasureFlickeringLuma** | `TSRMeasureFlickeringLuma.usf` | 直前フレームの luma 振動を測定 |
| **DilateVelocity** | `TSRDilateVelocity.usf` | velocity buffer をマックス膨張させ前景を優先 |
| **DecimateHistory** | `TSRDecimateHistory.usf` | 高解像ヒストリを resolve サイズへ間引き |
| **ClosestOccluder** | `TSRClosestOccluder.ush` | 最近接ブロッカー深度を抽出 |
| **DepthVelocityAnalysis** | `TSRDepthVelocityAnalysis.ush` | 深度差から disocclusion 判定 |
| **DetectThinGeometry** | `TSRDetectThinGeometry.usf` | 1 ピクセル幅構造検出 |
| **RejectShading** | `TSRRejectShading.usf` | 履歴 vs 現フレームの shading 比較。CNN 風 conv で per-tile 判定 |
| **MeasureCoverage** | `TSRMeasureCoverage.usf` | 有効サンプル被覆率 |
| **WeightRelaxation** | `TSRWeightRelaxation.usf` | 重み平滑化 |
| **SpatialAntiAliasing** | `TSRSpatialAntiAliasing.usf` | 空間 AA（フォールバック / aux） |
| **UpdateHistory** | `TSRUpdateHistory.usf` | 履歴累積（quality permutation × 4） |
| **ResolveHistory** | `TSRResolveHistory.usf` | 表示解像度へリサンプル |

各パスは **R11G11B10 履歴 + 16-bit VALU** で帯域 / 演算最適化。

### 2.1 Reprojection Field（[Epic 2022]）

各出力ピクセルから「最も信頼できる前フレーム位置」を:
1. dilated velocity から候補座標を取得
2. ClosestOccluder から最近接深度を採用
3. depth-velocity 一致性で重みづけ

→ 透明体の背景 / シルエット周辺で誤再投影を抑制。

### 2.2 Shading Rejection（核心）

TAA の Variance Clipping は近傍統計のみ。TSR は:

```
diff = abs(currentLuma - reprojectedHistory) / max(currentLuma, history, eps)
spatialFilter = ConvolutionNetwork(diff in 3x3 or 5x5)
rejectionMask = (spatialFilter > threshold)
```

`TSRConvolutionNetwork.ush` で 4 段の reduce / blur / sigmoid 風処理。タイル 32×32 を groupshared で完結（VRAM ラウンドトリップ無し）。

### 2.3 Flickering Heuristic

`r.TSR.ShadingRejection.Flickering=1` で有効化。

```
periodFrames = r.TSR.ShadingRejection.Flickering.Period (default 2)
if (luma が periodFrames 周期で振動している) → flickering 領域
    ghost を許容しつつ luma 振幅範囲内でブレンド
```

モアレ・極微細メッシュ・raster grid 干渉で発生する flicker を抑制。

### 2.4 Thin Geometry Detection

ワイヤ / 髪 / グリッドなど **1 ピクセル幅未満の構造**は通常 reproject すると消失:
- `TSRDetectThinGeometry.usf` で「現フレームに有るが履歴に無い」高周波を検出
- 該当ピクセルは新フレーム重み高、ghost 許容

### 2.5 History Sample Count

`r.TSR.History.SampleCount=16`（最大 32）が「1 出力ピクセルあたりの仮想累積サンプル数」上限。シェーディング rejection 後は `r.TSR.ShadingRejection.SampleCount=2` までリセット → 急速にリビルド。

### 2.6 History ScreenPercentage

`r.TSR.History.ScreenPercentage=100〜200`。**200% は Nyquist-Shannon 条件を満たす唯一の設定**で、ヒストリ内の高周波情報を完全に保持できる（Epic がドキュメントで明記）。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `PostProcess/TemporalSuperResolution.cpp` | TSR 全 pass オーケストレーション、CVar |
| `TemporalSuperResolution/TSRCommon.ush` | 共通定数、サンプリング |
| `TemporalSuperResolution/TSRColorSpace.ush` | YCoCg / 履歴エンコード |
| `TemporalSuperResolution/TSRConvolutionNetwork.ush` | shading rejection conv |
| `TemporalSuperResolution/TSRReprojectionField.ush` | 再投影座標生成 |
| `TemporalSuperResolution/TSRKernels.ush` | フィルタカーネル |

### 3.2 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.AntiAliasingMethod` | (Project) | 4 = TSR |
| `r.TSR.History.SampleCount` | 16.0 | 履歴サンプル数上限（最大 32） |
| `r.TSR.History.ScreenPercentage` | 100.0 | 履歴解像度倍率（200=Nyquist 完全） |
| `r.TSR.History.R11G11B10` | 1 | 履歴を R11G11B10 で保存 |
| `r.TSR.History.UpdateQuality` | 3 | UpdateHistory shader permutation |
| `r.TSR.WaveOps` | 1 | wave intrinsics 使用 |
| `r.TSR.WaveSize` | 0 | 0=auto, 16/32/64 |
| `r.TSR.16BitVALU` | 1 | fp16 VALU |
| `r.TSR.16BitVALU.AMD` | 1 | AMD 上書き |
| `r.TSR.16BitVALU.Nvidia` | 0 | Nvidia 上書き（fp16 が遅い世代対策） |
| `r.TSR.ShadingRejection.SampleCount` | 2.0 | rejection 後の履歴サンプル数 |
| `r.TSR.ShadingRejection.Flickering` | 1 | flicker 抑制 |
| `r.TSR.ShadingRejection.Flickering.Period` | 2.0 | 検出周期（フレーム） |
| `r.TSR.ShadingRejection.Flickering.FrameRateCap` | 60 | 自動 period 調整の上限 fps |
| `r.TSR.ShadingRejection.Flickering.MaxParallaxVelocity` | 10 | 視差オクルージョン許容速度 |
| `r.TSR.ShadingRejection.TileOverscan` | 3 | タイル境界余白 |
| `r.TSR.AlphaChannel` | -1 | alpha 伝播（-1=projet 設定追従） |
| `r.TSR.Support.LensDistortion` | 1 | レンズ歪み LUT サポート |

### 3.3 History Slice Sequence

`FTSRHistorySliceSequence`（`TemporalSuperResolution.cpp:1516-`）が複数フレーム履歴の rolling index を管理:
- 単一スライス: `FrameStorageCount=1`
- 多スライス + 周期: 「直近フレーム」と「数フレーム前のリザレクション用」を分離保持
- **Resurrection**: 急峻な disocclusion 後にしばらく前のヒストリへ巻き戻して復元

### 3.4 GPU 性能特性

- 1080p TSR: 1.5〜2 ms (RTX 3070 想定)
- shading rejection が最重い段（タイル convolution × 全画面）
- Wave64 / fp16 / R11G11B10 の組合わせで PS4Pro 級でも実用

---

## 4. 近似・省略の差分

| 項目 | 理想 | UE TSR | 影響 |
|------|------|-------|------|
| Reprojection 精度 | per-pixel 完全 | dilated velocity + closest occluder | 透過物背景でわずかにずれ |
| Shading rejection | 完全 NN | conv 風（数 op） | 訓練済 NN ほどの判別力なし |
| Thin geometry | 1 px 未満完全保持 | 検出 → 重み増 | 完全消失せず微サンプルは依然 missed |
| 履歴精度 | fp32 | R11G11B10 (default) | bright HDR で banding 微残 |
| Flickering 検出 | 任意周期 | period=2 frame デフォルト | period 不一致モアレで誤検 |

---

## 5. パラメータと CVar

§3.2 にまとめ済み。

| 用途 | 推奨 |
|------|------|
| 最高品質 | `r.TSR.History.ScreenPercentage=200` + `r.TSR.History.SampleCount=32` |
| 軽量化 | `r.TSR.WaveOps=1` + `r.TSR.16BitVALU=1` + `r.TSR.History.R11G11B10=1` |
| ghost が酷い | `r.TSR.ShadingRejection.SampleCount=1.0`（rejection 後の累積を強くリセット） |

---

## 6. 代替手法との比較

| 手法 | upsample | shading rejection | UE 採用 |
|------|---------|-----------------|--------|
| TAA (Karis 2014) | 1x | Variance Clipping | UE4 |
| TAAU | 〜1.5x | Variance Clipping | UE4.23 |
| **TSR (Epic 2022)** | **〜3x** | **Conv Network 風 + Flickering** | **UE5 標準** |
| DLSS 2/3 | 〜3x | NN | プラグイン |
| FSR 2/3 | 〜2x | 解析 + 履歴 | プラグイン |
| XeSS | 〜3x | NN (Intel) | プラグイン |

### TSR の独自性

- **オープン実装**で全パス読める
- **Nanite / Lumen と同期**: subpixel triangle や lumen noise を前提に最適化
- **モバイル向け 16bit VALU パス**で iOS/Android でも動作

---

## 7. 参考資料

- [S31] Epic 2022 "Temporal Super Resolution" / Lyra ポストモーテム
- Epic 公式ドキュメント "Temporal Super Resolution in Unreal Engine 5"
- Karis 2014 → [[aa_taa]]
- 関連: [[nanite_visibility_buffer]]（subpixel triangle が TSR と相性）

---

## 8. 相談用フック

- **理解度チェック**:
  - TSR が TAA と決定的に違う点 → shading rejection を独立 pass で行う
  - History.ScreenPercentage=200 の意味 → Nyquist-Shannon 条件で履歴の高周波が完全保存
  - Flickering Heuristic の対象 → モアレ / Nanite サブピクセル ジオメトリ
- **コード深掘り候補**:
  - `TSRRejectShading.usf` の conv 構造
  - `TSRConvolutionNetwork.ush` の reduce / sigmoid
  - `TSRUpdateHistory.usf` の DIM_UPDATE_QUALITY 4 種
  - `FTSRHistorySliceSequence` の rolling index 設計
- **未読箇所**:
  - Resurrection の発火条件
  - WaveSize 16/32/64 自動選択ロジック
  - LensDistortion LUT 連携の VALU コスト
- **次の派生**:
  - TAA → [[aa_taa]]
  - DLSS / FSR プラグイン統合（GTemporalUpscaler 差替え）
  - Nanite との相性 → [[nanite_visibility_buffer]]
