# Translucency ソースマップ

- 対象: 半透明描画（Forward Shading + Separate Translucency + OIT + Distortion）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[20_translucency_overview]]

`RenderTranslucency()` が深度ソート済みプリミティブを Back→Front に Forward Shading で描画。
TranslucencyLightingVolume（3D テクスチャ）で間接光を近似、DOF 前後でパスを分割する。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/` |
| メイン | `TranslucentRendering.h/.cpp` |
| ライティング | `TranslucentLightingShaders.h/.cpp` |
| OIT | `OIT/OITParameters.h`, `OIT/OIT.cpp` |
| 半透明 BasePass | `BasePassRendering.cpp` (`FTranslucentBasePassMeshProcessor`) |
| シェーダー | `Engine/Shaders/Private/TranslucentLightingShaders.usf`, `OITShaders/*.usf` |

---

## ファイル → クラス対応

### メインパス

| ファイル | 主要関数 / enum | 役割 | 参照 |
|---------|---------------|------|------|
| `TranslucentRendering.h/.cpp` | `RenderTranslucency()`, `RenderTranslucencyInner()`, `ETranslucencyPass::Type` | 半透明パスのオーケストレーション・Separate RT 管理 | [[Reference/ref_translucent_rendering]], [[Details/a_translucent_rendering]] |
| `BasePassRendering.cpp` | `FTranslucentBasePassMeshProcessor` | Forward Shading で半透明マテリアル評価 | 同 |

### ライティングボリューム

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `TranslucentLightingShaders.h/.cpp` | `RenderTranslucencyLightingVolume()`, `InjectTranslucencyLightingVolume()`, `FilterTranslucencyVolume()` | 3D テクスチャに間接光注入・3D ガウシアンブラー | [[Reference/ref_translucent_lighting]], [[Details/b_translucent_lighting]] |

### OIT（Order-Independent Translucency）

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `OIT/OIT.cpp` | `AccumulateOIT()`, `CompositeOIT()`, `FOITParameters` | Weighted Blended / Pixel Linked List | [[Details/c_oit]] |

### Distortion

| ファイル | 主要関数 | 役割 |
|---------|--------|------|
| `DistortionRendering.cpp` | `RenderDistortion()` | 屈折効果（ガラス等） |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─[A] RenderTranslucencyLightingVolume        TranslucentLightingShaders.cpp
  │       for each Local Light:
  │         InjectTranslucencyLightingVolume()  → 3D Volume に注入
  │       FilterTranslucencyVolume()            → 3D Gaussian ブラー
  │
  └─[B] RenderTranslucency                      TranslucentRendering.cpp
        │
        ├─ RenderTranslucencyInner(TPT_StandardTranslucency)
        │    FTranslucentBasePassMeshProcessor  BasePassRendering.cpp
        │    → Forward Shading（TranslucencyVolume サンプル）
        │    → SceneColor に加算 or AccumulateOIT
        │
        ├─ [DOF / MotionBlur]
        │
        ├─ RenderTranslucencyInner(TPT_TranslucencyAfterDOF)
        │    → Separate Translucency RT（低解像度可）に描画
        │
        ├─ CompositeOIT                         OIT.cpp（OIT 有効時）
        │
        └─ RenderDistortion                     DistortionRendering.cpp
```

---

## ETranslucencyPass（TranslucentRendering.h）

| enum | 用途 |
|------|------|
| `TPT_StandardTranslucency` | 通常（Before DOF） |
| `TPT_TranslucencyAfterDOF` | Separate Translucency（DOF 後） |
| `TPT_TranslucencyAfterDOFModulate` | DOF 後乗算 |
| `TPT_TranslucencyAfterMotionBlur` | MotionBlur 後 |
| `TPT_AllTranslucency` | 内部分類用 |

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.SeparateTranslucency` | `TranslucentRendering.cpp` |
| `r.SeparateTranslucencyScreenPercentage` | 同 |
| `r.TranslucencyLightingVolumeDim` | `TranslucentLightingShaders.cpp` |
| `r.TranslucencyVolumeBlur` | 同 |
| `r.OIT.SortedPixels` | `OIT.cpp` |
| `r.OIT.WeightedBlended` | 同 |
| `r.ParallelTranslucency` | `TranslucentRendering.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_translucent_rendering]] | RenderTranslucency |
| Reference | [[Reference/ref_translucent_lighting]] | TranslucencyLightingVolume |
| Details | [[Details/a_translucent_rendering]] | Forward Shading / Separate |
| Details | [[Details/b_translucent_lighting]] | Volume Inject / Filter |
| Details | [[Details/c_oit]] | OIT 実装 |

---

## ue5-dive 起点

- 「半透明の Forward Shading」 → `BasePassRendering.cpp:FTranslucentBasePassMeshProcessor`
- 「TranslucencyVolume の注入」 → `TranslucentLightingShaders.cpp:InjectTranslucencyLightingVolume`
- 「Separate Translucency の RT 作成」 → `TranslucentRendering.cpp:RenderTranslucency`（`TPT_TranslucencyAfterDOF` 分岐）
- 「OIT の Composite」 → `OIT.cpp:CompositeOIT`
- 「Distortion（屈折）」 → `DistortionRendering.cpp:RenderDistortion`
