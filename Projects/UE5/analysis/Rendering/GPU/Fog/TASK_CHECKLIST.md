# Fog GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_fog_gpu_overview.md` … Height Fog / Volumetric Fog GPU 全処理実行順 + CPU対応表

---

## Fog GPU シェーダー（グループ別）

### a: HeightFog（2本）
- [x] `a_HeightFog/detail_height_fog.md`  … 指数関数的 Height Fog の全画面合成パス
- [x] `a_HeightFog/ref_height_fog.md`     … ExponentialPixelMain / HeightFogVertexShader エントリポイント

### b: VolumetricFog（2本）
- [x] `b_VolumetricFog/detail_volumetric_fog.md`  … Froxel 単位の In-Scattering / Extinction 積分パス
- [x] `b_VolumetricFog/ref_volumetric_fog.md`     … MaterialSetupCS / LightScatteringCS / FinalIntegrationCS エントリポイント

---

合計: 概要 1 + Fog シェーダー 4 = **5 ファイル** ✅
