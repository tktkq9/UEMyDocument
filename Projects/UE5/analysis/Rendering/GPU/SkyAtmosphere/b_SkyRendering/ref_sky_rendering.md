# GPU b Ref: SkyRendering シェーダーリファレンス

- シェーダー: `SkyAtmosphere.usf`
- CPU 対応: [[a_sky_atmosphere]]
- 上位: [[01_skyatmosphere_gpu_overview]]

---

## エントリポイント

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `SkyAtmosphereVS` | 851 | VS | フルスクリーン三角形生成（VertexBuffer なし）|
| `RenderSkyAtmosphereRayMarchingPS` | 905 | PS | 空のレイマーチング描画 |
| `RenderSkyAtmosphereDebugPS` | 1705 | PS | デバッグ可視化 |
| `RenderSkyAtmosphereEditorHudPS` | 1854 | PS | エディタ HUD オーバーレイ |

---

## SkyAtmosphereVS 入出力

| パラメータ | 方向 | 型 | 説明 |
|-----------|------|-----|------|
| `VertexId` | 入力 | `uint : SV_VertexID` | 0/1/2 の三角形頂点 ID |
| `Position` | 出力 | `float4 : SV_POSITION` | クリップ空間位置 |
| `StartDepthZ` | グローバル | `float` | 深度テスト用の Z 値 |

---

## RenderSkyAtmosphereRayMarchingPS 入出力

| パラメータ | 方向 | 型 | 説明 |
|-----------|------|-----|------|
| `SVPos` | 入力 | `float4 : SV_POSITION` | スクリーン位置 |
| `OutLuminance` | 出力 | `float4 : SV_Target0` | RGB=輝度, A=透過率（グレースケール）|
| `SampleIndex` | 入力（MSAA 時）| `uint : SV_SampleIndex` | MSAA サンプルインデックス |

---

## LUT サンプリング関数

| 関数 | ファイル | 説明 |
|------|---------|------|
| `GetSkyRadiance()` | `SkyAtmosphereCommon.ush` | SkyViewLUT から指定方向の空輝度を取得 |
| `GetSkyLuminanceWithAerialPerspective()` | `SkyAtmosphere.usf` | AerialPerspective 込みの空輝度 |
| `GetAtmosphereTransmittance()` | `SkyAtmosphereCommon.ush` | TransmittanceLUT をサンプリング |

---

## グローバルパラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `StartDepthZ` | `float` | VS の深度値 |
| `DepthReadDisabled` | `uint` | 深度読み取り無効フラグ |
| `bPropagateAlphaNonReflection` | `uint` | Alpha 伝播フラグ（Reflection Capture 向け）|

---

## パーミュテーション

| マクロ | 値 | 説明 |
|--------|-----|------|
| `FASTSKY_ENABLED` | 0/1 | 高速スカイ（SkyViewLUT サンプリングのみ）|
| `FASTAERIALPERSPECTIVE_ENABLED` | 0/1 | 高速 AerialPerspective |
| `SOURCE_DISK_ENABLED` | 0/1 | 太陽・月ディスク描画 |
| `SECOND_ATMOSPHERE_LIGHT_ENABLED` | 0/1 | 第2ライト（月等）|
| `SAMPLE_OPAQUE_SHADOW` | 0/1 | 不透明シャドウのサンプリング |
| `SAMPLE_CLOUD_SHADOW` | 0/1 | クラウドシャドウのサンプリング |
| `SAMPLE_CLOUD_SKYAO` | 0/1 | クラウド Sky AO |
| `VIRTUAL_SHADOW_MAP` | 0/1 | VSM 対応 |
| `MSAA_SAMPLE_COUNT` | 1/2/4/8 | MSAA 対応 |
| `COLORED_TRANSMITTANCE_ENABLED` | 0 | デュアルソースブレンド（未使用）|
| `RENDERSKY_ENABLED` | 1 | メイン描画パスの識別マクロ |

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.SkyAtmosphere.FastSky` | `FASTSKY_ENABLED` 切り替え |
| `r.SkyAtmosphere.RenderSkyAtmosphere` | スカイ描画の有効/無効 |
| `r.SkyAtmosphere.SampleCountMin` | レイマーチ最低サンプル数 |
| `r.SkyAtmosphere.SampleCountMax` | レイマーチ最大サンプル数 |
| `r.SkyAtmosphere.DistanceToSampleCountMax` | 最大サンプル数適用距離 |
