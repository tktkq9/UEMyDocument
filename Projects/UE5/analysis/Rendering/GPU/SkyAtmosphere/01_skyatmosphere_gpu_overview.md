# GPU: SkyAtmosphere シェーダー全体概要

- 上位: [[01_gpu_overview]]
- CPU 対応: [[18_skyatmosphere_overview]]

---

## 概要

UE5 の Sky / Atmosphere GPU パスは物理ベースの大気散乱モデルを実装する。  
LUT（Look Up Table）を事前生成してメモリに保持し、毎フレームの描画コストを削減する。  
すべて `SkyAtmosphere.usf` 1ファイルにまとめられており、マクロパーミュテーションで各パスを切り替える。

---

## GPU シェーダーグループ対応表

| グループ | 主要エントリポイント | 用途 |
|---------|-----------------|------|
| **a: LUTBuild** | `RenderTransmittanceLutCS`<br>`RenderMultiScatteredLuminanceLutCS`<br>`RenderSkyViewLutCS`<br>`RenderDistantSkyLightLutCS`<br>`RenderCameraAerialPerspectiveVolumeCS` | 大気散乱 LUT の事前計算（CS）|
| **b: SkyRendering** | `SkyAtmosphereVS`<br>`RenderSkyAtmosphereRayMarchingPS` | 空のレイマーチングによる最終描画（VS/PS）|
| **c: SkyLight** | `RenderDistantSkyLightLutCS`<br>+ ConvolveSkyLight 系 | Sky Light キャプチャ・Sky AO |

---

## GPU データフロー

```
【LUT ビルド（非同期 Compute で事前計算）】
RenderSkyAtmosphereLookUpTables()
  ├─ RenderTransmittanceLutCS         → TransmittanceLUT (Texture2D)
  ├─ RenderMultiScatteredLuminanceLutCS → MultiScattLUT (Texture2D)
  ├─ RenderSkyViewLutCS               → SkyViewLUT (Texture2D)
  ├─ RenderDistantSkyLightLutCS       → DistantSkyLight LUT（Sky Light用）
  └─ RenderCameraAerialPerspectiveVolumeCS → AerialPerspective LUT (Texture3D)

【本描画（Opaque パス後）】
RenderSkyAtmosphere()
  └─ SkyAtmosphereVS + RenderSkyAtmosphereRayMarchingPS
       → SceneColor に加算合成

【Sky Light キャプチャ（条件付き）】
CaptureOrUpdateSkyOnce()
  → DistantSkyLight LUT からキューブマップへ変換
```

---

## パーミュテーション一覧

| マクロ | 意味 |
|--------|------|
| `TRANSMITTANCE_PASS` | Transmittance LUT ビルドパス |
| `MULTISCATT_PASS` | MultiScattering LUT ビルドパス |
| `SKYVIEWLUT_PASS` | SkyView LUT ビルドパス |
| `SKYLIGHT_PASS` | DistantSkyLight LUT ビルドパス |
| `FASTSKY_ENABLED` | 近似スカイ（品質/速度トレードオフ）|
| `FASTAERIALPERSPECTIVE_ENABLED` | 近似 AerialPerspective |
| `MULTISCATTERING_APPROX_SAMPLING_ENABLED` | 多重散乱近似サンプリング |
| `SECOND_ATMOSPHERE_LIGHT_ENABLED` | 第2ライト（月光等）対応 |
| `SAMPLE_OPAQUE_SHADOW` | 不透明シャドウのサンプリング |
| `RENDERSKY_ENABLED` | 空の描画有効 |

---

## 各グループ詳細

| グループ | 詳細 | リファレンス |
|---------|------|------------|
| a: LUTBuild | [[a_LUTBuild/detail_lut_build]] | [[a_LUTBuild/ref_lut_build]] |
| b: SkyRendering | [[b_SkyRendering/detail_sky_rendering]] | [[b_SkyRendering/ref_sky_rendering]] |
| c: SkyLight | [[c_SkyLight/detail_sky_light]] | [[c_SkyLight/ref_sky_light]] |

---

## CPU 側対応

| CPU ドキュメント | 対応 GPU グループ |
|---------------|----------------|
| [[a_sky_atmosphere]] — USkyAtmosphereComponent / FSkyAtmosphereRenderSceneInfo | グループ a, b |
| [[b_sky_light]] — USkyLightComponent / Sky AO | グループ c |
