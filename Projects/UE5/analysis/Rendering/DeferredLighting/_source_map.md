# DeferredLighting ソースマップ

- 対象: Deferred Lighting（GBuffer → SceneColor 直接光）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[16_deferred_lighting_overview]]

`RenderLights()` が `FSortedLightSetSceneInfo` の分類に従い、
Clustered Deferred / Standard Deferred / MegaLights に振り分けて SceneColor へ加算する。

---

## ソースパス

| 対象 | パス |
|------|------|
| ライトパイプライン統括 | `Engine/Source/Runtime/Renderer/Private/LightRendering.h/.cpp` |
| 個別ライト描画 | `Engine/Source/Runtime/Renderer/Private/DeferredLightRendering.h/.cpp` |
| Clustered Deferred | `Engine/Source/Runtime/Renderer/Private/ClusteredDeferredShadingRenderer.h/.cpp` |
| Light Function | `Engine/Source/Runtime/Renderer/Private/LightFunctionRendering.cpp` |
| ライトシーン情報 | `Engine/Source/Runtime/Renderer/Private/LightSceneInfo.h/.cpp` |
| GPU シェーダー | `Engine/Shaders/Private/DeferredLightPixelShaders.usf`, `DeferredLightVertexShaders.usf` |

---

## ファイル → クラス対応

### 統括

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `LightRendering.cpp` | `RenderLights()`:1520, `GatherLightsAndComputeLightGrid()`, `GatherLightsForView()` | ライト分類・シャドウ統合・パス選択 | [[Reference/ref_light_rendering]], [[Details/a_light_rendering]] |

### 個別ライト描画

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `DeferredLightRendering.cpp` | `RenderLight()`, `FDeferredLightPS`, 境界ジオメトリ描画 | UnbatchedLight 用 PS パス | [[Reference/ref_light_rendering]] |
| `LightFunctionRendering.cpp` | `FLightFunctionPS` | Light Function マテリアル適用 | [[Details/d_light_functions]] |

### Clustered/Tiled

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `ClusteredDeferredShadingRenderer.cpp` | `AddClusteredDeferredShadingPass()`, `FClusteredDeferredLightingCS` | Simple+Batched ライトを CS 一括処理 | [[Reference/ref_clustered_tiled]], [[Details/b_clustered_tiled]] |
| `TiledDeferredLightRendering.cpp` | Tiled Deferred CS（フォールバック） | タイル別 CS 処理 | 同 |

### シーン情報

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `LightSceneInfo.h` | `FLightSceneInfo`, `FSortedLightSceneInfo`, `FSortedLightSetSceneInfo` | ライトのシーン表現・ソート構造 | [[Reference/ref_light_params]] |
| `LightSceneInfo.cpp` | `AddLightInteraction`, シャドウキャッシュ連携 | プリミティブ相互作用 | — |

---

## `FSortedLightSetSceneInfo` のインデックス境界

```
[0 .. SimpleLightsEnd)            SimpleLights（パーティクル等）
[SimpleLightsEnd .. ClusteredEnd) Clustered Deferred 対象
[ClusteredEnd .. UnbatchedStart)  Standard Deferred（影なし）
[UnbatchedStart .. end)           Standard Deferred（影あり）
MegaLightsLightStart              MegaLights 対象開始
```

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─ GatherLightsAndComputeLightGrid()                LightRendering.cpp
  │    ├─ GetLightOcclusionType → Shadowmap/Raytraced/MegaLights/MegaLightsVSM
  │    ├─ SortedLights を SortKey でソート
  │    └─ InjectLightsIntoGrid → LightDataBuffer / ClusteredLightGrid
  │
  └─ RenderLights()                                   LightRendering.cpp:1520
       ├─[A] Substrate ステンシルパス
       │     AddSubstrateStencilPass
       │
       ├─[B] Shadow 前処理
       │     ShadowSceneRenderer.RenderVirtualShadowMapProjectionMaskBits
       │     FProjectedShadowInfo::RenderProjectedShadow（UnbatchedLights 用）
       │
       ├─[C] Clustered Deferred                       ClusteredDeferredShadingRenderer.cpp
       │     AddClusteredDeferredShadingPass
       │      └─ [0..ClusteredEnd) を CS 一括処理
       │
       ├─[D] Standard Deferred（影なし）
       │     for [StandardDeferredStart..UnbatchedLightStart):
       │       RenderLight(Light, Shadow=nullptr)     DeferredLightRendering.cpp
       │        └─ 球/コーン/全画面クワッド + DeferredLightPixelShaders.usf
       │
       └─[E] UnbatchedLights（影あり）
             for [UnbatchedLightStart..end):
               ├─ RTShadow or 従来 ShadowMask
               ├─ IScreenSpaceDenoiser::DenoiseShadows
               └─ RenderLight(Light, ShadowMaskTexture)
```

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.AllowDepthBoundsTest` | `LightRendering.cpp` |
| `r.ClusteredDeferredShading` | `ClusteredDeferredShadingRenderer.cpp` |
| `r.AllowSimpleLights` | `LightRendering.cpp` |
| `r.Shadow.Denoiser` | `LightRendering.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_light_rendering]] | RenderLights / RenderLight |
| Reference | [[Reference/ref_clustered_tiled]] | Clustered / Tiled Deferred |
| Reference | [[Reference/ref_light_params]] | FLightSceneInfo / FSortedLightSetSceneInfo |
| Details | [[Details/a_light_rendering]] | パイプライン全体 |
| Details | [[Details/b_clustered_tiled]] | Clustered/Tiled 実装詳細 |
| Details | [[Details/c_reflection_skylight]] | 反射・SkyLight 統合 |
| Details | [[Details/d_light_functions]] | Light Function 適用 |

---

## ue5-dive 起点

- 「ライトがどの経路で描画されるか」 → `LightRendering.cpp:RenderLights` + SortKey 境界値
- 「Clustered と Standard の切り替え条件」 → `ShouldUseClusteredDeferredShading()` + `AreLightsInLightGrid()`
- 「シャドウマスクがライトに適用される位置」 → `DeferredLightRendering.cpp:RenderLight` 引数 `ShadowMaskTexture`
- 「MegaLights 対象判定」 → `GetLightOcclusionType` + `MegaLightsLightStart`
- 「新ライト種別の追加」 → `FLightSceneInfo` + `FSortedLightSceneInfo::SortKey` 更新
