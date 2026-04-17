# Deferred Lighting GPU シェーダードキュメント チェックリスト

## 概要
- [ ] `01_deferred_lighting_gpu_overview.md` … Deferred Lighting GPU 全処理実行順 + CPU対応表

---

## Deferred Lighting GPU シェーダー（グループ別）

### a: PointSpotLight（2本）
- [ ] `a_PointSpotLight/detail_point_spot.md`  … Point / Spot ライトの Clustered / Tiled Deferred 処理
- [ ] `a_PointSpotLight/ref_point_spot.md`     … DeferredLightPixelShaders / TiledDeferredLighting エントリポイント

### b: DirectionalLight（2本）
- [ ] `b_DirectionalLight/detail_directional.md`  … ディレクショナルライトの全画面ライティングパス
- [ ] `b_DirectionalLight/ref_directional.md`     … DirectionalLightPS / AtmosphereLightScattering エントリポイント

### c: LightFunctions（2本）
- [ ] `c_LightFunctions/detail_light_functions.md`  … ライト関数マテリアルのライトへの適用
- [ ] `c_LightFunctions/ref_light_functions.md`     … LightFunctionPixelShader / InjectionLightFunction エントリポイント

### d: SkyLight（2本）
- [ ] `d_SkyLight/detail_sky_light.md`  … SkyLight キューブマップ・Lumen Sky Light 合成
- [ ] `d_SkyLight/ref_sky_light.md`     … SkyLightDiffuse / ReflectionCapture エントリポイント

---

合計: 概要 1 + Deferred Lighting シェーダー 8 = **9 ファイル**
