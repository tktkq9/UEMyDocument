# Screen Space AO ドキュメント タスクチェックリスト

**ソース**: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\`  
**GPU 対応**: [[GPU/SSAO/TASK_CHECKLIST]]

---

## 概要ファイル

- [x] `22_ssao_overview.md` — SSAO / Lumen Short Range AO の全体フロー・切り替え条件

---

## Phase 1: Details（2本）

- [x] `Details/a_ssao.md`               — `AddAmbientOcclusionPasses()` フロー・半球サンプリング・空間ブラー
- [x] `Details/b_composition_lighting.md` — `CompositionLighting` による AO / GI 合成パス・PostProcessVolume との連携

---

## Phase 2: Reference（2本）

- [x] `Reference/ref_ssao.md`               — `FAmbientOcclusionParameters` / `FAmbientOcclusionSetup` / SSAO シェーダークラス群
- [x] `Reference/ref_composition_lighting.md` — `FPostProcessingInputs` / `FCompositionLighting` フレームワーク

---

## Phase 3: コード実行フロー追加

- [x] `22_ssao_overview.md`                — `CompositionLighting.ProcessAfterOcclusion()` → SSAO / Lumen AO 分岐フロー追加
- [x] `Details/a_ssao.md`                  — SSAO Setup → Horizon → Blur → Composite フロー
- [x] `Details/b_composition_lighting.md`  — `AddPostProcessingPasses()` 内での AO 統合フロー

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
| `AmbientOcclusionRendering.h/.cpp` | SSAO パス定義・`AddAmbientOcclusionPasses()` |
| `CompositionLighting.h/.cpp` | AO / GI 合成フレームワーク |
| `PostProcessAmbientOcclusion.h/.cpp` | PostProcess 側の AO 統合 |
