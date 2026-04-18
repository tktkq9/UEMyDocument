# Fog ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\`  
**GPU 対応**: [[GPU/Fog/TASK_CHECKLIST]]

---

## 概要ファイル

- [x] `19_fog_overview.md` — Height Fog / Volumetric Fog の全体フロー・両者の使い分け

---

## Phase 1: Details（2本）

- [x] `Details/a_height_fog.md`      — `RenderFog()` フロー・指数関数的 Height Fog の CPU 側パラメータ設定
- [x] `Details/b_volumetric_fog.md`  — `RenderVolumetricFog()` フロー・Voxel グリッド管理・Froxel 積分

---

## Phase 2: Reference（2本）

- [x] `Reference/ref_fog_rendering.md`    — `FExponentialHeightFogSceneInfo` / `FFogUniformParameters` / `FHeightFogVS`
- [x] `Reference/ref_volumetric_fog.md`   — `FVolumetricFogIntegrationParameters` / `FVolumetricFogLocalAmbientOcclusion`

---

## Phase 3: コード実行フロー追加

- [x] `19_fog_overview.md`          — `RenderFog()` + `RenderVolumetricFog()` 実行順・RDG パス構成フロー追加
- [x] `Details/a_height_fog.md`     — `FExponentialHeightFogSceneInfo` 更新 → `RenderFog()` → HeightFogPS フロー
- [x] `Details/b_volumetric_fog.md` — Froxel 生成 → In-Scattering 注入 → 積分 → 合成フロー

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
| `FogRendering.h/.cpp` | Height Fog レンダリング・`RenderFog()` |
| `VolumetricFogRendering.h/.cpp` | Volumetric Fog Froxel・積分・`RenderVolumetricFog()` |
| `VolumetricCloudRendering.h/.cpp` | 体積雲（Volumetric Fog と連携）|
