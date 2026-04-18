# Deferred Decals GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_decals_gpu_overview.md` … Deferred Decals GPU 全処理実行順 + CPU対応表

---

## Decals GPU シェーダー（グループ別）

### a: DeferredDecal（2本）
- [x] `a_DeferredDecal/detail_deferred_decal.md`  … GBuffer 後置デカール投影（Translucent/Stain/Normal/Emissive）
- [x] `a_DeferredDecal/ref_deferred_decal.md`     … DeferredDecal MainVS / FPixelShaderInOut_MainPS エントリポイント

### b: DBuffer（2本）
- [x] `b_DBuffer/detail_dbuffer.md`  … Pre-GBuffer DBuffer デカール合成（マテリアル前合成方式）
- [x] `b_DBuffer/ref_dbuffer.md`     … WriteDBufferData / ApplyDBufferData / FDBufferData エントリポイント

---

合計: 概要 1 + Decals シェーダー 4 = **5 ファイル** ✅
