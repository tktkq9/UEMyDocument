# Base Pass GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_basepass_gpu_overview.md` … Base Pass / GBuffer GPU 全処理実行順 + CPU対応表

---

## Base Pass GPU シェーダー（グループ別）

### a: Opaque（2本）
- [x] `a_Opaque/detail_opaque.md`  … 非Nanite不透明メッシュの GBuffer 書き込み（VS + PS）
- [x] `a_Opaque/ref_opaque.md`     … BasePassVertexShader#Main / BasePassPixelShader エントリポイント

### b: NaniteMaterial（2本）
- [x] `b_NaniteMaterial/detail_nanite_material.md`  … NaniteExportGBuffer / ShadeBinning による GBuffer 生成
- [x] `b_NaniteMaterial/ref_nanite_material.md`     … ShadeBinBuildCS / EmitSceneDepthPS エントリポイント

### c: Substrate（2本）
- [x] `c_Substrate/detail_substrate.md`  … Substrate マテリアルレイヤーの GBuffer エンコード
- [x] `c_Substrate/ref_substrate.md`     … SubstrateConvertToLegacyGBuffer / SubstratePixelExport エントリポイント

---

合計: 概要 1 + Base Pass シェーダー 6 = **7 ファイル** ✅
