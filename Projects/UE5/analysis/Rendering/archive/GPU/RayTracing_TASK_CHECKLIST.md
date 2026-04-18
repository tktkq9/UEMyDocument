# Ray Tracing GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_raytracing_gpu_overview.md` … Ray Tracing GPU 全処理実行順 + CPU対応表

---

## Ray Tracing GPU シェーダー（グループ別）

### a: Core（2本）
- [x] `a_Core/detail_core.md`  … TLAS / BLAS 構築・更新（DXR / VK_RT）
- [x] `a_Core/ref_core.md`     … BuildRayTracingScene / UpdateRayTracingGeometry エントリポイント

### b: Shadow（2本）
- [x] `b_Shadow/detail_shadow.md`  … HW Ray Traced Shadow（Any-Hit / Miss shader）
- [x] `b_Shadow/ref_shadow.md`     … RayTracedShadowRGS / RayTracedShadowAHS エントリポイント

### c: AO（2本）
- [x] `c_AO/detail_ao.md`  … Ray Traced Ambient Occlusion（短距離 AO レイ + デノイズ）
- [x] `c_AO/ref_ao.md`     … RayTracedAO RGS / RayTracedAODenoiser エントリポイント

### d: Reflection（2本）
- [x] `d_Reflection/detail_reflection.md`  … HW Ray Traced Reflections（Lumen HWRT バックエンド）
- [x] `d_Reflection/ref_reflection.md`     … LumenHWRTReflection RGS / LumenHWRTReflectionCHS エントリポイント

### e: Translucency（2本）
- [x] `e_Translucency/detail_translucency.md`  … Ray Traced Translucency（半透明 Hit Shader）
- [x] `e_Translucency/ref_translucency.md`     … RayTracedTranslucency RGS / CHS エントリポイント

---

合計: 概要 1 + Ray Tracing シェーダー 10 = **11 ファイル**
