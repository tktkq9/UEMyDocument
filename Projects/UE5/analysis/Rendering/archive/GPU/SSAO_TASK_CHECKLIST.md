# Screen Space AO GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_ssao_gpu_overview.md` … SSAO / Lumen AO GPU 全処理実行順 + CPU対応表

---

## SSAO GPU シェーダー（グループ別）

### a: SSAO（2本）
- [x] `a_SSAO/detail_ssao.md`  … Screen Space Ambient Occlusion（GTAO: HorizonSearch + Integrate + Filter）
- [x] `a_SSAO/ref_ssao.md`     … MainSetupPS / HorizonSearchCS / GTAOCombinedCS / GTAOTemporalFilterCS エントリポイント

### b: LumenAO（2本）
- [x] `b_LumenAO/detail_lumen_ao.md`  … Lumen Short Range AO + Screen Space Bent Normal 計算
- [x] `b_LumenAO/ref_lumen_ao.md`     … ScreenSpaceShortRangeAOCS エントリポイント

---

合計: 概要 1 + SSAO シェーダー 4 = **5 ファイル** ✅
