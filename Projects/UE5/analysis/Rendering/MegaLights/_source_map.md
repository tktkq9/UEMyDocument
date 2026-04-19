# MegaLights ソースマップ

- 対象: 多数ダイナミックライトの確率的サンプリングシステム（UE5.4〜）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[06_megalights_overview]]

タイル分類 → 確率的サンプリング → RT or VSM シャドウ → Resolve → Denoise → TranslucencyVolume の 6 段構成。
固定コストで数千ライトを扱えるようにする。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/MegaLights/` |
| シェーダー | `Engine/Shaders/Private/MegaLights/*.usf` |
| 呼び出し元 | `DeferredShadingRenderer.cpp:3318`（`RenderMegaLights`） |

---

## ファイル → クラス対応

### エントリ / パイプライン

| ファイル | 主要クラス / 関数 / enum | 役割 | 参照 |
|---------|----------------------|------|------|
| `MegaLights.h/.cpp` | `MegaLights::IsEnabled()`, `GetMegaLightsMode()`, `UseHardwareRayTracing()`, `UseVolume()`, `IsMarkingVSMPages()`, `GetDownsampleFactorXY()`, `EMegaLightsMode` | 有効条件判定・動作モード | [[a_megalights_pipeline]], [[ref_megalights_core]] |
| `MegaLights.cpp:1959` | `RenderMegaLights()` | 全ビューオーケストレーション | 同 |
| `MegaLights.cpp:1920` | `RenderMegaLightsViewContext()` | 1 ビュー分の全パス実行 | 同 |
| `MegaLightsInternal.h` | `FMegaLightsFrameTemporaries`, 内部共通型 | フレームリソース一時保持 | 同 |
| `MegaLightsViewState.h` | テンポラル蓄積ステート | History ブレンド | — |

### サンプリング / シャドウ

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `MegaLightsSampling.cpp` | `FGenerateLightSamplesCS` | 確率的ライトサンプリング | [[b_megalights_sampling]] |
| `MegaLightsRayTracing.cpp` | `MegaLights::RayTraceLightSamples()` | HW RT シャドウ（`EnabledRT`） | [[c_megalights_shadow]] |
| `MegaLights.cpp` | `MegaLights::MarkVSMPages()` | VSM ページマーキング（`EnabledVSM`） | 同 |

### Resolve / Denoise

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `MegaLightsResolve.cpp` | `FMegaLightsResolveCS` | BRDF 評価・全サンプル合算 | [[d_megalights_resolve]] |
| `MegaLightsDenoising.cpp` | Temporal / Spatial フィルタ | テンポラルデノイズ | 同 |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()
  │
  └─ RenderMegaLights                              DeferredShadingRenderer.cpp:3318
       │                                            MegaLights.cpp:1959
       └─ for each ViewIndex:
           ├─ RenderMegaLightsViewContext          MegaLights.cpp:1920
           │   │
           │   ├─[A] DownsampleSceneDepth / WorldNormal   半解像度テクスチャ
           │   │
           │   ├─[B] GenerateLightSamples                 MegaLightsSampling.cpp
           │   │     FGenerateLightSamplesCS
           │   │     → LightSamples テクスチャ（ライト ID + 方向）
           │   │     Far Field GDF 対応（UseFarField）
           │   │
           │   ├─[C] VisibilityTest（シャドウ）
           │   │   ├─[EnabledRT]  RayTraceLightSamples    MegaLightsRayTracing.cpp
           │   │   └─[EnabledVSM] MarkVSMPages            MegaLights.cpp
           │   │
           │   ├─[D] Resolve                              MegaLightsResolve.cpp
           │   │     FMegaLightsResolveCS
           │   │     BRDF 評価 → 全サンプル合算・正規化
           │   │
           │   ├─[E] Denoise                              MegaLightsDenoising.cpp
           │   │   ├─ Temporal: History ブレンド
           │   │   └─ Spatial:  空間フィルタ
           │   │
           │   └─[F] [Volume 有効] TranslucencyVolume 注入
           │
           └─[HairStrands 有効] RenderMegaLightsViewContext(ViewContextHair, ...)
```

---

## EMegaLightsMode

| enum | 用途 |
|------|------|
| `Disabled` | 無効 |
| `EnabledRT` | HW Ray Tracing シャドウ使用 |
| `EnabledVSM` | Virtual Shadow Maps シャドウ使用 |

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.MegaLights` | `MegaLights.cpp`（0=無効, 1=VSM, 2=RT） |
| `r.MegaLights.HardwareRayTracing` | `MegaLightsRayTracing.cpp` |
| `r.MegaLights.Volume` | `MegaLights.cpp` |
| `r.MegaLights.FarField` | `MegaLightsSampling.cpp` |
| `r.MegaLights.Denoiser` | `MegaLightsDenoising.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[ref_megalights_core]] | コア型定義 |
| Details | [[a_megalights_pipeline]] | パイプライン全体・EMegaLightsInput |
| Details | [[b_megalights_sampling]] | 確率的サンプリング |
| Details | [[c_megalights_shadow]] | RT / VSM シャドウ統合 |
| Details | [[d_megalights_resolve]] | Resolve / Denoise / Volume |

---

## ue5-dive 起点

- 「MegaLights のエントリ」 → `DeferredShadingRenderer.cpp:3318:RenderMegaLights` → `MegaLights.cpp:1959`
- 「RT と VSM の分岐」 → `GetMegaLightsMode()` + `EMegaLightsMode`
- 「ライト確率サンプリング」 → `MegaLightsSampling.cpp:FGenerateLightSamplesCS`
- 「RT シャドウ経路」 → `MegaLightsRayTracing.cpp:RayTraceLightSamples`
- 「VSM ページマーキング経路」 → `MegaLights.cpp:MarkVSMPages`（VSM と連動）
- 「半透明への照明」 → `r.MegaLights.Volume` + TranslucencyVolume 注入
