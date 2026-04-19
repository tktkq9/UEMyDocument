# Decals ソースマップ

- 対象: Deferred Decals（GBuffer Decal + DBuffer Decal + Emissive + AO Decal）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[15_decals_overview]]

投影 OBB で GBuffer / DBuffer / SceneColor / AO バッファにマテリアルを塗るシステム。
`EDecalRenderStage` でステージ分岐し、DBuffer は BasePass 前、GBuffer は BasePass 後・Lighting 前に実行する。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/` |
| 共有ヘッダ | `DecalRenderingCommon.h`, `DecalRenderingShared.h/.cpp` |
| GBuffer Decal | `DeferredDecalRendering.cpp` |
| DBuffer Decal | `DBufferDecalRendering.cpp` |
| 可視性 | `DecalVisibilityTaskData.cpp` |
| シェーダー | `Engine/Shaders/Private/DeferredDecal.usf`, `DBufferDecal.usf` |

---

## ファイル → クラス対応

### 共有・可視性

| ファイル | 主要クラス / enum | 役割 | 参照 |
|---------|----------------|------|------|
| `DecalRenderingCommon.h` | `EDecalRenderStage`, `EDecalBlendMode`, `FDecalBlendDesc` | ステージ/ブレンド分類 | [[Reference/ref_deferred_decal]] |
| `DecalRenderingShared.h/.cpp` | `FVisibleDecal`, `FTransientDecalRenderData`, `GetDecalRenderStage()` | 可視性結果・描画データ | 同 |
| `DecalVisibilityTaskData.cpp` | `FDecalVisibilityTaskData::Launch()` | 非同期デカール可視判定 | 同 |

### GBuffer Decals

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `DeferredDecalRendering.cpp` | `RenderDeferredDecals()` | BeforeLighting / Emissive / AO ステージ | [[Reference/ref_deferred_decal]] |

### DBuffer Decals

| ファイル | 主要関数 / 構造体 | 役割 | 参照 |
|---------|----------------|------|------|
| `DBufferDecalRendering.cpp` | `RenderDBufferDecals()`, `FDBufferTextures`, `FDBufferData`, `ApplyDBufferDecal()` | BeforeBasePass の DBuffer A/B/C 書き込み | [[Reference/ref_dbuffer]] |

---

## EDecalRenderStage（DecalRenderingCommon.h）

| enum | 対応パス | 書き込み先 |
|------|--------|----------|
| `BeforeBasePass` | RenderDBufferDecals | DBuffer A/B/C |
| `BeforeLighting` | RenderDeferredDecals | GBuffer |
| `Emissive` | RenderDeferredDecals | SceneColor |
| `AmbientOcclusion` | RenderDeferredDecals | AO バッファ |
| `Mobile` / `MobileBeforeLighting` | モバイル専用 | — |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─[InitViews]
  │   FDecalVisibilityTaskData::Launch           非同期デカール可視判定
  │
  ├─[BeforeBasePass] RenderDBufferDecals         DBufferDecalRendering.cpp
  │     └─ DBuffer A/B/C（BaseColor / Normal / RoughnessMetallic）に投影書き込み
  │
  ├─[BasePass]
  │   各メッシュで ApplyDBufferDecal → BaseColor/Normal/RM に乗算合成
  │
  ├─[BeforeLighting] RenderDeferredDecals        DeferredDecalRendering.cpp
  │     └─ ステンシルで OBB 外を除外 → GBuffer チャンネルを上書き
  │
  ├─[Emissive] RenderDeferredDecals              SceneColor に加算
  │
  └─[AmbientOcclusion] RenderDeferredDecals      AO バッファ書き込み
```

---

## EDecalBlendMode / BlendMode 対応表

| BlendMode | Stage | 対象 |
|-----------|-------|------|
| Translucent | BeforeLighting | GBuffer（色/法線/RM）|
| Stain | BeforeLighting | GBuffer（色） |
| Normal | BeforeLighting | GBuffer（法線） |
| Emissive | Emissive | SceneColor |
| DBuffer Translucent | BeforeBasePass | DBuffer |
| DBuffer Normal | BeforeBasePass | DBuffer |
| DBuffer Color | BeforeBasePass | DBuffer |
| AmbientOcclusion | AmbientOcclusion | AO バッファ |

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.DBuffer` | `DBufferDecalRendering.cpp` |
| `r.Decal.StencilBackFaceSelection` | `DeferredDecalRendering.cpp` |
| `r.DecalFadeDuration` | `DecalRenderingShared.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_deferred_decal]] | FDecalBlendDesc / FVisibleDecal / DecalVisibilityTaskData |
| Reference | [[Reference/ref_dbuffer]] | FDBufferTextures / FDBufferData / ApplyDBufferDecal |

---

## ue5-dive 起点

- 「DBuffer と GBuffer Decal の切替」 → `EDecalBlendMode` + `GetDecalRenderStage()`
- 「DBuffer の合成位置」 → BasePass 内 `ApplyDBufferDecal`
- 「ステンシル投影カリング」 → `DeferredDecalRendering.cpp:RenderDeferredDecals` + `r.Decal.StencilBackFaceSelection`
- 「非同期可視判定」 → `DecalVisibilityTaskData.cpp:FDecalVisibilityTaskData::Launch`
- 「Emissive Decal」 → `EDecalRenderStage::Emissive` 分岐
