# GPUScene GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_gpuscene_gpu_overview.md` … GPUScene GPU 全処理実行順 + CPU対応表

---

## GPUScene GPU シェーダー（グループ別）

### a: Upload（2本）
- [x] `a_Upload/detail_upload.md`  … PrimitiveSceneData / InstanceSceneData のGPUバッファ転送
- [x] `a_Upload/ref_upload.md`     … GPUSceneSetInstancePrimitiveIdCS エントリポイント

### b: InstanceCulling（2本）
- [x] `b_InstanceCulling/detail_instance_culling.md`  … GPU Driven Instance Culling DispatchPasses
- [x] `b_InstanceCulling/ref_instance_culling.md`     … InstanceCullBuildInstanceIdBufferCS / ClearIndirectArgInstanceCountCS エントリポイント

### c: BuildInfo（2本）
- [x] `c_BuildInfo/detail_build_info.md`  … SceneBuildInfo / LightData GPU転送・インデックス構築
- [x] `c_BuildInfo/ref_build_info.md`     … LightDataBuffer / GPUScene SOA レイアウト エントリポイント

---

合計: 概要 1 + GPUScene シェーダー 6 = **7 ファイル** ✅
