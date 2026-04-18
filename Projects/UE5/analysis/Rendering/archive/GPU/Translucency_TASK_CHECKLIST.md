# Translucency GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_translucency_gpu_overview.md` … Translucency GPU 全処理実行順 + CPU対応表

---

## Translucency GPU シェーダー（グループ別）

### a: TranslucentForward（2本）
- [x] `a_TranslucentForward/detail_translucent_forward.md`  … 半透明メッシュのフォワードシェーディング（深度ソート）
- [x] `a_TranslucentForward/ref_translucent_forward.md`     … BasePassPixelShader(Translucent) / TranslucentShadingVS エントリポイント

### b: TranslucentLighting（2本）
- [x] `b_TranslucentLighting/detail_translucent_lighting.md`  … 半透明オブジェクト用ライティングボリューム注入・補間
- [x] `b_TranslucentLighting/ref_translucent_lighting.md`     … InjectTranslucentVolumeLighting / FilterTranslucentVolume エントリポイント

### c: OIT（2本）
- [x] `c_OIT/detail_oit.md`  … Order-Independent Translucency（Weighted Blended OIT）
- [x] `c_OIT/ref_oit.md`     … OITAccumulate / OITComposite エントリポイント

---

合計: 概要 1 + Translucency シェーダー 6 = **7 ファイル**
