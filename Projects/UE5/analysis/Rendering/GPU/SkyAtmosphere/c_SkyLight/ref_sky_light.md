# GPU c Ref: SkyLight シェーダーリファレンス

- シェーダー: `SkyAtmosphere.usf`, `ReflectionEnvironmentShaders.usf`
- CPU 対応: [[b_sky_light]]
- 上位: [[01_skyatmosphere_gpu_overview]]

---

## エントリポイント

| 関数名 | ファイル | 行番号 | タイプ | 説明 |
|--------|---------|--------|--------|------|
| `RenderDistantSkyLightLutCS` | `SkyAtmosphere.usf` | 1404 | CS | 大気輝度 LUT 生成 |
| `ConvolveSpecularSkyLightCS` | `ReflectionEnvironmentShaders.usf` | — | CS | スペキュラキューブマップ Mip フィルタ |
| `ComputeSkyEnvMapDiffuseIrradianceCS` | `ReflectionEnvironmentShaders.usf` | — | CS | Diffuse Irradiance（球面調和）計算 |

---

## RenderDistantSkyLightLutCS パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `DistantSkyLightLutUAV` | `RWTexture2D<float4>` | 出力: 大気輝度テーブル |
| `LutSizeAndInvSize` | `float4` | LUT 解像度（xy）と逆数（zw）|

内部でサンプリングする LUT:
- `TransmittanceLutTexture`（透過率）
- `MultiScatteredLuminanceLutTexture`（多重散乱）

---

## Sky Light キューブマップ関連パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `InTexture` | `TextureCube` | 入力キューブマップ |
| `OutTextureMipColor` | `RWTexture2DArray<float4>` | 出力 Mip スライス |
| `MipIndex` | `int` | 現在の Mip レベル |
| `NumMips` | `int` | 総 Mip 数 |
| `RoughnessMip` | `float` | このMip に対応する Roughness |

---

## SkyAtmosphereCommon.ush 主要関数（LUT サンプリング）

| 関数 | 説明 |
|------|------|
| `GetAtmosphereTransmittance(WorldPos, WorldDir, ...)` | TransmittanceLUT から透過率を取得 |
| `GetMultipleScattering(WorldPos, SunZenithCosAngle)` | MultiScattLUT から多重散乱輝度を取得 |
| `SkyAtmosphereCommonApplyAerialPerspective(...)` | AerialPerspective Volume を適用 |

---

## FSkyAtmosphereRenderContext（CPU 側構造体）

`SkyAtmosphereRendering.h:71`

| メンバ | 型 | 説明 |
|--------|-----|------|
| `bSkyAtmospherePresentInScene` | `bool` | SkyAtmosphere が存在するか |
| `bShouldRenderSkyAtmosphere` | `bool` | 実際にレンダリングするか |
| `SkyAtmosphereSceneInfo` | `FSkyAtmosphereRenderSceneInfo*` | シーン情報へのポインタ |

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.SkyAtmosphere.SkyLightCapture` | Sky Light のリアルタイムキャプチャ |
| `r.SkyAtmosphere.DistantSkyLightLUT.SampleCount` | DistantSkyLight LUT のサンプル数 |
| `r.SkyLight.RealTimeCapture` | リアルタイム Sky Light キャプチャ |
| `r.ReflectionEnvironment` | ReflectionEnvironment（Sky Light Specular）の有効/無効 |
