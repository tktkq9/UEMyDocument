# MegaLights ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\MegaLights\`

---

## 概要ファイル

- [x] `06_megalights_overview.md` — アーキテクチャ・フロー・主要クラス・CVar

---

## Phase 1: Details（4本）

- [x] `Details/a_megalights_pipeline.md` — TileClassify → Sampling → Shadow → Resolve → Denoise 全体パイプライン
- [x] `Details/b_megalights_sampling.md` — 確率的ライトサンプリング・ライストリスト構築・Far Field GDF
- [x] `Details/c_megalights_shadow.md`   — RT シャドウ（HW RT）/ VSM ページマーキング 両モードの統合
- [x] `Details/d_megalights_resolve.md`  — サンプル合算・テンポラル蓄積・TranslucencyVolume 照明注入

---

## Phase 2: Reference（3本）

- [x] `Reference/ref_megalights_core.md`      — `EMegaLightsMode` / `FMegaLightsFrameTemporaries` / `FMegaLightsVolume` / `FMegaLightsViewContext`
- [x] `Reference/ref_megalights_sampling.md`  — `FMegaLightsParameters` / `FMegaLightsVolumeParameters` / `EMegaLightsInput`
- [x] `Reference/ref_megalights_denoising.md` — `FMegaLightsViewState` / `FResources` / ヒストリフロー

---

## Phase 3: コード実行フロー追加

- [x] `06_megalights_overview.md` — RenderMegaLights() 内部フロー（全パス呼び出し順序）追加
- [x] `Details/a_megalights_pipeline.md` — TileClassify→Stochastic→Shadow→Resolve フロー
- [x] `Details/b_megalights_sampling.md` — GetMegaLightsMode() 判定 + ライストリスト構築フロー
- [x] `Details/c_megalights_shadow.md`   — EnabledRT / EnabledVSM 条件分岐フロー
- [x] `Details/d_megalights_resolve.md`  — Resolve → Denoiser → Volume 注入フロー

---

## 進捗サマリ

| フェーズ | 合計 | 完了 | 残り |
|---------|------|------|------|
| 概要 | 1 | 1 | 0 |
| Phase 1: Details | 4 | 4 | 0 |
| Phase 2: Reference | 3 | 3 | 0 |
| Phase 3: Flow | 5 | 5 | 0 |
| **合計** | **13** | **13** | **0** |

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `MegaLights.h/.cpp` | エントリポイント・TileClassify・全体オーケストレーション |
| `MegaLightsInternal.h` | 内部共通型定義 |
| `MegaLightsSampling.cpp` | 確率的ライトサンプリング |
| `MegaLightsRayTracing.cpp` | HW RT シャドウパス |
| `MegaLightsResolve.cpp` | サンプル合算・リゾルブ |
| `MegaLightsDenoising.cpp` | テンポラルデノイズ |
| `MegaLightsViewState.h` | フレーム間ステート（テンポラル蓄積用）|
