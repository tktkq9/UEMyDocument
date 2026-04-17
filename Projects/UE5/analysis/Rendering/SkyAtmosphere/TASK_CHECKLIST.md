# Sky / Atmosphere ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\`  
**GPU 対応**: [[GPU/SkyAtmosphere/TASK_CHECKLIST]]

---

## 概要ファイル

- [x] `18_skyatmosphere_overview.md` — Sky Atmosphere の全体フロー・LUT 構築・Sky Light 統合

---

## Phase 1: Details（2本）

- [x] `Details/a_sky_atmosphere.md` — `RenderSkyAtmosphere()` フロー・LUT 生成（Transmittance / SkyView / AerialPerspective）
- [x] `Details/b_sky_light.md`      — Sky Light キャプチャ・`ConvolveSpecularSkyLight()` / Distance Field AO 統合

---

## Phase 2: Reference（2本）

- [x] `Reference/ref_sky_atmosphere.md` — `FSkyAtmosphereRenderSceneInfo` / `FSkyAtmosphereSceneProxy` / `FSkyAtmosphereUniformShaderParameters`
- [x] `Reference/ref_sky_light.md`      — `FSkyLightSceneProxy` / `FSkyLightParameters` / `FReflectionUniformParameters`

---

## Phase 3: コード実行フロー追加

- [x] `18_skyatmosphere_overview.md` — `RenderSkyAtmosphereMainView()` → LUT パス → Sky View → Aerial Perspective フロー追加
- [x] `Details/a_sky_atmosphere.md`  — 各 LUT パスのパラメータ依存・RDG パス構成フロー
- [x] `Details/b_sky_light.md`       — SkyLight キャプチャ → Mip フィルタリング → ConvolveSpecular フロー

---

## 進捗サマリ

| フェーズ | 合計 | 完了 | 残り |
|---------|------|------|------|
| 概要 | 1 | 1 | 0 |
| Phase 1: Details | 2 | 2 | 0 |
| Phase 2: Reference | 2 | 2 | 0 |
| Phase 3: Flow | 3 | 3 | 0 |
| **合計** | **8** | **8** | **0** |

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `SkyAtmosphereRendering.h/.cpp` | Sky Atmosphere LUT・メインレンダリング |
| `SkyPassRendering.h/.cpp` | Sky Sphere パス |
| `SkyAtmosphereRendering.h` | FSkyAtmosphereRenderSceneInfo クラス定義 |
| `ReflectionEnvironmentCapture.h/.cpp` | SkyLight キャプチャ・Convolve |
