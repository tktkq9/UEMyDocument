# Sky / Atmosphere GPU シェーダードキュメント チェックリスト

## 概要
- [ ] `01_skyatmosphere_gpu_overview.md` … Sky / Atmosphere GPU 全処理実行順 + CPU対応表

---

## Sky / Atmosphere GPU シェーダー（グループ別）

### a: LUTBuild（2本）
- [ ] `a_LUTBuild/detail_lut_build.md`  … Transmittance / Scattering / MultiScattering LUT 生成
- [ ] `a_LUTBuild/ref_lut_build.md`     … SkyAtmosphereLUT / TransmittanceLUT / SkyViewLUT エントリポイント

### b: SkyRendering（2本）
- [ ] `b_SkyRendering/detail_sky_rendering.md`  … Physical Sky / Sky Sphere の全画面パス描画
- [ ] `b_SkyRendering/ref_sky_rendering.md`     … SkyAtmosphereMainView / RenderSkyAtmosphereEditorNotifications エントリポイント

### c: SkyLight（2本）
- [ ] `c_SkyLight/detail_sky_light.md`  … Sky Light キャプチャ・ConvolveSkyLight / DistanceField AO
- [ ] `c_SkyLight/ref_sky_light.md`     … ConvolveSpecularSkyLight / DistanceFieldAO エントリポイント

---

合計: 概要 1 + Sky/Atmosphere シェーダー 6 = **7 ファイル**
