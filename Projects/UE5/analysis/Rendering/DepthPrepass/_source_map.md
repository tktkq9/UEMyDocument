# DepthPrepass ソースマップ

- 対象: Depth Pre-pass（Early-Z）+ Velocity Buffer
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[13_depthprepass_overview]]

BasePass 前に深度を確定する EarlyZ パスと、TSR/TAA/MotionBlur 用の Velocity Buffer 描画。
両者は同じ EMeshPass 基盤（`FDepthPassMeshProcessor` / `FVelocityMeshProcessor`）上に乗る。

---

## ソースパス

| 対象 | パス |
|------|------|
| Depth Pre-pass | `Engine/Source/Runtime/Renderer/Private/DepthRendering.h/.cpp` |
| Velocity | `Engine/Source/Runtime/Renderer/Private/VelocityRendering.h/.cpp` |
| Nanite Depth Resolve | `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteDepth.cpp` |
| 呼び出し元 | `Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp`, `SceneRendering.cpp` |
| GPU シェーダー | `Engine/Shaders/Private/DepthOnlyVertexShader.usf`, `DepthOnlyPixelShader.usf`, `VelocityShader.usf` |

---

## ファイル → クラス対応

### Depth Pre-pass

| ファイル | 主要クラス / enum | 役割 | 参照 |
|---------|----------------|------|------|
| `DepthRendering.h` | `EDepthDrawingMode`（DDM_*）, `FDepthPassMeshProcessor`, `TDepthOnlyVS<>`, `FDepthOnlyPS` | 描画モード enum・プロセッサ・シェーダー | [[Reference/ref_depth_rendering]] |
| `DepthRendering.cpp` | `FDeferredShadingSceneRenderer::RenderPrePass()`, `AddDepthPrePass()` | RDG Pass 登録・Nanite / 非Nanite 分岐 | [[Details/a_depth_rendering]] |

### Velocity Buffer

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `VelocityRendering.h` | `FVelocityMeshProcessor`, `FVelocityVS/PS` | Velocity 用プロセッサ・シェーダー | [[Reference/ref_velocity_rendering]] |
| `VelocityRendering.cpp` | `AddVelocityPass()`, `RenderVelocities()`, `RenderTranslucencyVelocities()` | Opaque / Translucent 別の Velocity 描画 | [[Details/b_velocity_rendering]] |

### Nanite 連携

| ファイル | 役割 |
|---------|------|
| `Nanite/NaniteDepth.cpp` | VisBuffer64 から SceneDepth への Resolve（`EmitDepth` CS）|
| `Nanite/NaniteVelocity.cpp` | Nanite 用 Velocity CS |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()             DeferredShadingRenderer.cpp
  │
  ├─[1] GetDepthPassInfo(Scene) → EarlyZPassMode 確定
  │
  ├─[2] RenderPrePass()                             DepthRendering.cpp
  │      ├─ [Nanite] Nanite::EmitDepth()            Nanite/NaniteDepth.cpp
  │      │   └─ VisBuffer64 → D24S8 Resolve CS
  │      ├─ [非Nanite Opaque] FDepthPassMeshProcessor
  │      │   └─ bUsePositionOnlyStream=true / TDepthOnlyVS<true>
  │      └─ [非Nanite Masked] FDepthPassMeshProcessor
  │          └─ FDepthOnlyPS で OpacityMask clip
  │
  └─[3] AddVelocityPass()                           VelocityRendering.cpp
         ├─ FVelocityMeshProcessor（Static / Skinned）
         ├─ [Nanite] Nanite::RenderVelocities()    Nanite/NaniteVelocity.cpp
         └─ R16G16_FLOAT Velocity へ書き込み
```

---

## EDepthDrawingMode（`DepthRendering.h:22`）

| enum | 意味 |
|------|------|
| `DDM_None` | PrePass 無効 |
| `DDM_NonMaskedOnly` | Opaque のみ |
| `DDM_AllOccluders` | Opaque + Masked（UseAsOccluder のみ）|
| `DDM_AllOpaque` | 全 Opaque（デフォルト、`r.EarlyZPass=3`）|
| `DDM_MaskedOnly` | Masked のみ |
| `DDM_AllOpaqueNoVelocity` | Velocity を除く全 Opaque |

---

## 主要 CVar

| CVar | 対応位置 |
|------|--------|
| `r.EarlyZPass` | `DepthRendering.cpp` |
| `r.EarlyZPassMovable` | `DepthRendering.cpp` |
| `r.ParallelPrePass` | `DepthRendering.cpp` |
| `r.VelocityOutputPass` | `VelocityRendering.cpp` |
| `r.BasePassOutputsVelocity` | `BasePassRendering.cpp`（BasePass 同時出力）|

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_depth_rendering]] | FDepthPassMeshProcessor |
| Reference | [[Reference/ref_velocity_rendering]] | FVelocityMeshProcessor |
| Details | [[Details/a_depth_rendering]] | Depth Pre-pass 実装詳細 |
| Details | [[Details/b_velocity_rendering]] | Velocity Buffer 実装詳細 |

---

## ue5-dive 起点

- 「PrePass が実際に何を描画しているか」 → `DepthRendering.cpp:RenderPrePass` + `r.EarlyZPass` 値
- 「Position-only stream の仕組み」 → `TDepthOnlyVS<true>` + `FVertexFactory` の Position-only 経路
- 「Velocity Buffer の座標系」 → `VelocityRendering.cpp` + `VelocityShader.usf`
- 「Nanite と非 Nanite の Depth 合成」 → `Nanite/NaniteDepth.cpp:EmitDepth`
