# GPU a Ref: SkyAtmosphere LUTBuild シェーダーリファレンス

- シェーダー: `SkyAtmosphere.usf`
- CPU 対応: [[a_sky_atmosphere]]
- 上位: [[01_skyatmosphere_gpu_overview]]

---

## シェーダーエントリポイント一覧

| CS 名 | 行番号 | 出力テクスチャ | スレッドグループ |
|-------|--------|------------|--------------|
| `RenderTransmittanceLutCS` | 1144 | `TransmittanceLutUAV` (R11G11B10 / RGBA8) | `[THREADGROUP_SIZE, THREADGROUP_SIZE, 1]` |
| `RenderMultiScatteredLuminanceLutCS` | 1200 | `MultiScatteredLuminanceLutUAV` | `[THREADGROUP_SIZE, THREADGROUP_SIZE, 1]` |
| `RenderSkyViewLutCS` | 1326 | `SkyViewLutUAV` | `[THREADGROUP_SIZE, THREADGROUP_SIZE, 1]` |
| `RenderDistantSkyLightLutCS` | 1404 | SkyLight 輝度テーブル | `[THREADGROUP_SIZE, THREADGROUP_SIZE, 1]` |
| `RenderCameraAerialPerspectiveVolumeCS` | 1510 | `AerialPerspectiveVolumeUAV` (Texture3D) | `[THREADGROUP_SIZE, THREADGROUP_SIZE, 1]` |

---

## パラメータ構造体

### SkyAtmosphere パラメータ（HLSL 変数）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `SkyAtmosphere.TransmittanceLutSizeAndInvSize` | `float4` | LUT 解像度（xy）と逆数（zw）|
| `SkyAtmosphere.TransmittanceSampleCount` | `uint` | 透過率 LUT の積分サンプル数 |
| `SkyAtmosphere.MultiScatteringSampleCount` | `uint` | 多重散乱 LUT のサンプル数 |
| `SkyAtmosphere.MultiScatteredLuminanceLutSizeAndInvSize` | `float4` | 多重散乱 LUT 解像度 |
| `Atmosphere.BottomRadiusKm` | `float` | 地球半径（km）|
| `Atmosphere.TopRadiusKm` | `float` | 大気最外層半径（km）|

---

## 物理ベース散乱関数

| 関数名 | ファイル | 説明 |
|--------|---------|------|
| `IntegrateSingleScatteredLuminance()` | `SkyAtmosphere.usf` | 単一散乱の輝度とオプティカルデプスを積分 |
| `UvToLutTransmittanceParams()` | `SkyAtmosphereCommon.ush` | UV → (高度, 仰角コサイン) に変換 |
| `UvToSkyViewLutDir()` | `SkyAtmosphereCommon.ush` | UV → スカイビュー方向ベクトルに変換 |
| `GetAtmosphereTransmittance()` | `SkyAtmosphereCommon.ush` | TransmittanceLUT をサンプリング |
| `GetMultipleScattering()` | `SkyAtmosphereCommon.ush` | MultiScattLUT をサンプリング |

---

## LUT テクスチャ仕様

| LUT | 解像度 | フォーマット（PC） | フォーマット（Mobile）|
|-----|--------|-----------------|---------------------|
| TransmittanceLUT | 256×64 | `PF_FloatR11G11B10` | `PF_B8G8R8A8` |
| MultiScattLUT | 32×32 | `PF_FloatR11G11B10` | `PF_B8G8R8A8` |
| SkyViewLUT | 192×108 | `PF_FloatR11G11B10` | `PF_B8G8R8A8` |
| AerialPerspectiveVolume | 32×32×16 | `PF_FloatRGBA` | `PF_FloatRGBA` |

---

## パーミュテーション

| マクロ | 値 | 意味 |
|--------|-----|------|
| `TRANSMITTANCE_PASS` | 1 | Transmittance LUT 生成パス |
| `MULTISCATT_PASS` | 1 | MultiScattering LUT 生成パス |
| `SKYVIEWLUT_PASS` | 1 | SkyView LUT 生成パス |
| `SKYLIGHT_PASS` | 1 | DistantSkyLight LUT 生成パス |
| `MULTISCATTERING_APPROX_SAMPLING_ENABLED` | 0/1 | 多重散乱の近似使用 |
| `HIGHQUALITY_MULTISCATTERING_APPROX_ENABLED` | 0/1 | 高品質多重散乱 |
| `FASTSKY_ENABLED` | 0/1 | 高速スカイ近似 |
| `FASTAERIALPERSPECTIVE_ENABLED` | 0/1 | 高速 AerialPerspective |

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.SkyAtmosphere` | SkyAtmosphere システム全体の有効/無効 |
| `r.SkyAtmosphere.FastSky` | `FASTSKY_ENABLED` を制御 |
| `r.SkyAtmosphere.AerialPerspective.FastApply` | `FASTAERIALPERSPECTIVE_ENABLED` を制御 |
| `r.SkyAtmosphere.TransmittanceLUT.SampleCount` | 透過率 LUT のサンプル数（精度）|
| `r.SkyAtmosphere.MultiScatteringLUT.HighQuality` | 多重散乱 LUT の品質 |
| `r.SkyAtmosphere.SampleCountMin` | 最低レイマーチサンプル数 |
