# Lumen GPU シェーダードキュメント チェックリスト

## 概要
- [x] `../01_gpu_overview.md` … GPU 全処理実行順 + CPU対応表（全体）
- [x] `01_lumen_gpu_overview.md` … Lumen GPU 全処理実行順 + CPU対応表

---

## Lumen GPU シェーダー（グループ別）

### a: Surface Cache / Card Capture（2本）
- [x] `a_SurfaceCache/detail_card_capture.md`   … Card Capture VS/PS/CS の役割・パス・RT構成
- [x] `a_SurfaceCache/ref_card_shaders.md`       … LumenCardVertexShader / PixelShader / ComputeShader / LumenSurfaceCache エントリポイント一覧

### b: Scene Lighting（4本）
- [x] `b_SceneLighting/detail_direct_lighting.md`  … Surface Cache への Direct Lighting 書き込み
- [x] `b_SceneLighting/ref_direct_lighting.md`     … LumenSceneDirectLighting 系シェーダー全エントリポイント
- [x] `b_SceneLighting/detail_radiosity.md`        … Radiosity（間接光バウンス）
- [x] `b_SceneLighting/ref_radiosity.md`           … LumenRadiosity 系シェーダー全エントリポイント

### c: Tracing（2本）
- [x] `c_Tracing/detail_sdf_tracing.md`  … Mesh SDF カリング + Scene SDF トレース
- [x] `c_Tracing/ref_sdf_tracing.md`     … LumenMeshSDFCulling / LumenScene 全エントリポイント

### d: Radiance Cache（2本）
- [x] `d_RadianceCache/detail_radiance_cache.md`  … Radiance Cache 更新フロー
- [x] `d_RadianceCache/ref_radiance_cache.md`     … LumenRadianceCache / LumenRadianceCacheUpdate / HW RT 版エントリポイント

### e: Diffuse GI（3本）
- [x] `e_DiffuseGI/detail_screen_probe_gather.md`  … Screen Probe 配置・トレース・Gather・テンポラル
- [x] `e_DiffuseGI/ref_screen_probe_gather.md`     … LumenScreenProbeGather / Tracing / Temporal / Filtering エントリポイント
- [x] `e_DiffuseGI/ref_restir_bent_normal.md`      … LumenReSTIRGather / LumenScreenSpaceBentNormal エントリポイント

### f: Reflections（3本）
- [x] `f_Reflections/detail_reflections.md`     … 反射トレース・Resolve・デノイズフロー
- [x] `f_Reflections/ref_reflection_tracing.md` … LumenReflections / ReflectionTracing / HW RT エントリポイント
- [x] `f_Reflections/ref_reflection_denoiser.md`… LumenReflectionDenoiser（Clear / Spatial / Temporal）エントリポイント

### g: Final Composite（2本）
- [x] `g_FinalComposite/detail_diffuse_indirect_composite.md` … DiffuseIndirectComposite の合成ロジック
- [x] `g_FinalComposite/ref_diffuse_indirect_composite.md`    … DiffuseIndirectComposite.usf エントリポイント・パーミュテーション一覧

---

合計: 概要 1 + Lumen シェーダー 18 = **19 ファイル**
