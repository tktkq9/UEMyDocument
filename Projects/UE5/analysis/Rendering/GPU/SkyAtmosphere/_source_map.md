# GPU SkyAtmosphere ソースマップ

- 対象: SkyAtmosphere GPU シェーダー（4 LUT ビルド + RayMarching 本描画 + SkyLight キャプチャ）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_skyatmosphere_gpu_overview]]

物理ベース大気散乱。Transmittance/MultiScattering/SkyView/AerialPerspective の LUT を事前計算し、
毎フレームのレイマーチングコストを削減。すべて `SkyAtmosphere.usf` 1 ファイル + マクロパーミュテーション。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/SkyAtmosphere.usf` |
| CPU | `Renderer/Private/SkyAtmosphereRendering.cpp` |

---

## ファイル → シェーダー対応

### LUT ビルド

| 主要エントリ | CPU 関数 | 役割 | 参照 |
|-----------|---------|------|------|
| `RenderTransmittanceLutCS` | `RenderSkyAtmosphereLookUpTables()` | Transmittance LUT（Texture2D） | [[detail_lut_build]] |
| `RenderMultiScatteredLuminanceLutCS` | 同上 | MultiScattering LUT（Texture2D） | 同 |
| `RenderSkyViewLutCS` | 同上 | SkyView LUT（Texture2D） | 同 |
| `RenderDistantSkyLightLutCS` | 同上 | DistantSkyLight LUT（SkyLight 用） | 同 |
| `RenderCameraAerialPerspectiveVolumeCS` | 同上 | AerialPerspective LUT（Texture3D） | 同 |

### 本描画

| 主要エントリ | CPU 関数 | 役割 | 参照 |
|-----------|---------|------|------|
| `SkyAtmosphereVS` + `RenderSkyAtmosphereRayMarchingPS` | `RenderSkyAtmosphere()` | 空のレイマーチング → SceneColor 加算 | [[detail_sky_rendering]] |

### SkyLight キャプチャ

| 主要エントリ | CPU 関数 | 役割 | 参照 |
|-----------|---------|------|------|
| `RenderDistantSkyLightLutCS` + ConvolveSkyLight 系 | `CaptureOrUpdateSkyOnce()` | DistantSkyLight LUT → キューブマップ | [[detail_sky_light]] |

---

## GPU データフロー

```
[LUT ビルド（非同期 Compute で事前計算）]
  RenderSkyAtmosphereLookUpTables()
    ├─ RenderTransmittanceLutCS              → TransmittanceLUT
    ├─ RenderMultiScatteredLuminanceLutCS    → MultiScattLUT
    ├─ RenderSkyViewLutCS                    → SkyViewLUT
    ├─ RenderDistantSkyLightLutCS            → DistantSkyLight LUT
    └─ RenderCameraAerialPerspectiveVolumeCS → AerialPerspective LUT (3D)

[本描画（Opaque 後）]
  RenderSkyAtmosphere()
    SkyAtmosphereVS + RenderSkyAtmosphereRayMarchingPS
    → SceneColor 加算合成

[SkyLight キャプチャ（条件付き）]
  CaptureOrUpdateSkyOnce()
    → DistantSkyLight LUT → キューブマップ変換
```

---

## パーミュテーション一覧

| マクロ | 意味 |
|--------|------|
| `TRANSMITTANCE_PASS` | Transmittance LUT ビルド |
| `MULTISCATT_PASS` | MultiScattering LUT ビルド |
| `SKYVIEWLUT_PASS` | SkyView LUT ビルド |
| `SKYLIGHT_PASS` | DistantSkyLight LUT ビルド |
| `FASTSKY_ENABLED` | 近似スカイ |
| `FASTAERIALPERSPECTIVE_ENABLED` | 近似 AerialPerspective |
| `MULTISCATTERING_APPROX_SAMPLING_ENABLED` | 多重散乱近似 |
| `SECOND_ATMOSPHERE_LIGHT_ENABLED` | 第 2 ライト（月光等） |
| `SAMPLE_OPAQUE_SHADOW` | 不透明シャドウサンプリング |
| `RENDERSKY_ENABLED` | 空描画有効 |

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_lut_build]] / [[detail_sky_rendering]] / [[detail_sky_light]] |
| Reference | [[ref_lut_build]] / [[ref_sky_rendering]] / [[ref_sky_light]] |

---

## ue5-dive 起点

- 「大気 LUT ビルド」 → `SkyAtmosphere.usf:Render*LutCS` + `RenderSkyAtmosphereLookUpTables()`
- 「空本描画」 → `SkyAtmosphere.usf:RenderSkyAtmosphereRayMarchingPS`
- 「AerialPerspective」 → `RenderCameraAerialPerspectiveVolumeCS`
- 「SkyLight キャプチャ」 → `CaptureOrUpdateSkyOnce()`
