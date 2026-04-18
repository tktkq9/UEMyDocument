# HZB（Hierarchical Z-Buffer）GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_hzb_gpu_overview.md` … HZB GPU 全処理実行順 + CPU対応表

---

## HZB GPU シェーダー（グループ別）

### a: HZBBuild（2本）
- [x] `a_HZBBuild/detail_hzb_build.md`  … SceneDepth からミップチェーン生成（各ミップを Reduce Min/Max）
- [x] `a_HZBBuild/ref_hzb_build.md`     … HZBBuildCS / HZBBuildPS エントリポイント

### b: OcclusionTest（2本）
- [x] `b_OcclusionTest/detail_occlusion_test.md`  … HZB を利用した GPU Occlusion Query / Feedback
- [x] `b_OcclusionTest/ref_occlusion_test.md`     … HZBTestPS / IsVisibleHZB / BoxCullFrustum エントリポイント

---

合計: 概要 1 + HZB シェーダー 4 = **5 ファイル** ✅
