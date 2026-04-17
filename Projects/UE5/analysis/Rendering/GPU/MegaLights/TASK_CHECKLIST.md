# MegaLights GPU シェーダードキュメント チェックリスト

## 概要
- [ ] `01_megalights_gpu_overview.md` … MegaLights GPU 全処理実行順 + CPU対応表

---

## MegaLights GPU シェーダー（グループ別）

### a: LightSampling（2本）
- [ ] `a_LightSampling/detail_light_sampling.md`  … レイベースのライトサンプリング（BRDF / Visibility）
- [ ] `a_LightSampling/ref_light_sampling.md`     … MegaLightsSampling / MegaLightsRayGen エントリポイント

### b: Denoising（2本）
- [ ] `b_Denoising/detail_denoising.md`  … MegaLights ノイズ除去（Spatial / Temporal フィルタ）
- [ ] `b_Denoising/ref_denoising.md`     … MegaLightsDenoiser / MegaLightsTemporal エントリポイント

### c: Composite（2本）
- [ ] `c_Composite/detail_composite.md`  … MegaLights 結果の SceneColor 合成
- [ ] `c_Composite/ref_composite.md`     … MegaLightsComposite / MegaLightsApply エントリポイント

---

合計: 概要 1 + MegaLights シェーダー 6 = **7 ファイル**
