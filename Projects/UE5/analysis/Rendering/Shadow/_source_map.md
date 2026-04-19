# Shadow ソースマップ

- 対象: 従来型シャドウ（Shadow Depth + Projection）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[17_shadow_overview]]
- 注意: **VSM は別フォルダ** [[VirtualShadowMaps/_source_map]]

`FProjectedShadowInfo` を中心に、CSM / Per-Object / Spot / Point Shadow を管理。
InitViews でセットアップ → RenderShadowDepthMaps → RenderLights 内の Projection で消費。

---

## ソースパス

| 対象 | パス |
|------|------|
| Shadow 統括・Projection | `Engine/Source/Runtime/Renderer/Private/ShadowRendering.h/.cpp` |
| Shadow セットアップ | `Engine/Source/Runtime/Renderer/Private/ShadowSetup.cpp` |
| Shadow Depth 描画 | `Engine/Source/Runtime/Renderer/Private/ShadowDepthRendering.h/.cpp` |
| Projection PS | `Engine/Source/Runtime/Renderer/Private/ShadowProjectionPixelShader.h` |
| Shadow Scene 管理 | `Engine/Source/Runtime/Renderer/Private/Shadows/ShadowSceneRenderer.h/.cpp` |
| Nanite Shadow | `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteShadowDepthRendering.cpp` |
| GPU シェーダー | `Engine/Shaders/Private/ShadowDepthVertexShader.usf`, `ShadowProjectionPixelShader.usf`, `ShadowProjectionCommon.ush` |

---

## ファイル → クラス対応

### 中核

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `ShadowRendering.h` | `FProjectedShadowInfo`, `FShadowCascadeSettings`, `TShadowProjectionPS<>` | シャドウの主要構造体・Projection PS | [[Reference/ref_projected_shadow_info]], [[Reference/ref_shadow_projection]] |
| `ShadowRendering.cpp` | `FDeferredShadingSceneRenderer::RenderShadowDepthMaps()`, `FProjectedShadowInfo::RenderDepth()`, `RenderProjectedShadow()` | Depth 描画・Projection 描画 | [[Reference/ref_shadow_rendering]], [[Details/b_shadow_depth]], [[Details/c_shadow_projection]] |

### セットアップ

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `ShadowSetup.cpp` | `SetupMeshDrawCommandsForShadowDepth()`, `CreatePerObjectProjectedShadow()`, `CreateWholeSceneProjectedShadow()`, `AllocateSubjectPrimitives()` | シャドウ種別分類・アトラスパッキング・プリミティブ収集 | [[Details/a_shadow_setup]] |

### Depth 描画

| ファイル | 主要クラス | 役割 |
|---------|----------|------|
| `ShadowDepthRendering.h/.cpp` | `FShadowDepthMeshProcessor`, `FShadowDepthVS`, `FShadowDepthPS` | ShadowDepth 用 MeshProcessor・深度専用シェーダー |
| `Nanite/NaniteShadowDepthRendering.cpp` | `Nanite::RenderShadowDepths()` | Nanite 経由の Shadow Depth |

### Shadow Scene（UE5.4+）

| ファイル | 主要クラス | 役割 |
|---------|----------|------|
| `Shadows/ShadowSceneRenderer.h/.cpp` | `FShadowSceneRenderer`, `ProcessInvalidations()` | シャドウシーン管理・VSM 連携（キャッシュ無効化） |
| `Shadows/ShadowScene.h/.cpp` | `FShadowScene` | シーン側の永続シャドウ情報 |

---

## FProjectedShadowInfo の生存期間

```
InitViews()
  └─ SetupMeshDrawCommandsForShadowDepth()           ShadowSetup.cpp
      for each FLightSceneInfo:
        Create{PerObject|WholeScene}ProjectedShadow
          → FProjectedShadowInfo を new
          → Atlas (X,Y,ResX,ResY) 割り当て
          → FVisibleLightInfo::AllProjectedShadows に追加

RenderShadowDepthMaps()                              ShadowRendering.cpp
  for each FProjectedShadowInfo:
    ├─ [Nanite] Nanite::RenderShadowDepths
    └─ [非Nanite] FShadowDepthMeshProcessor + FShadowDepthVS/PS
    → Shadow Depth Atlas (R32 or D24)

RenderLights() 内 UnbatchedLights ループ             LightRendering.cpp
  for each Light with Shadow:
    FProjectedShadowInfo::RenderProjectedShadow      ShadowRendering.cpp
      └─ TShadowProjectionPS<PCFSamples>
          GBuffer.Depth と Shadow Depth 比較 → Shadow Mask (R8)
          PCF / PCSS フィルタ
    → RenderLight(Light, ShadowMaskTexture)
```

---

## シャドウ種別

| 種別 | 対象ライト | クラス / 関数 |
|------|---------|-------------|
| Whole Scene / CSM | Directional | `CreateWholeSceneProjectedShadow` + `FShadowCascadeSettings` |
| Spot | Spot Light | `CreatePerObjectProjectedShadow` (Cone) |
| Point | Point Light | キューブマップ（6 面）|
| Per-Object | 動的オブジェクト | `CreatePerObjectProjectedShadow` |
| Cached | Stationary Light | `r.Shadow.CacheWholeSceneShadows` で静的ジオメトリ保持 |

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.Shadow.CSM.MaxCascades` | `ShadowSetup.cpp` |
| `r.Shadow.MaxResolution` | `ShadowSetup.cpp` |
| `r.Shadow.CSM.TransitionScale` | `ShadowRendering.cpp` |
| `r.Shadow.DistanceScale` | `ShadowSetup.cpp` |
| `r.Shadow.PerObject` | `ShadowSetup.cpp` |
| `r.Shadow.CacheWholeSceneShadows` | `ShadowSetup.cpp` |
| `r.Shadow.FilterMethod` | `ShadowRendering.cpp`（PCF/PCSS）|

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_projected_shadow_info]] | FProjectedShadowInfo |
| Reference | [[Reference/ref_shadow_rendering]] | RenderShadowDepthMaps |
| Reference | [[Reference/ref_shadow_projection]] | RenderProjectedShadow / FShadowProjectionPS |
| Details | [[Details/a_shadow_setup]] | 種別分類・アトラスパッキング |
| Details | [[Details/b_shadow_depth]] | Shadow Depth 描画詳細 |
| Details | [[Details/c_shadow_projection]] | PCF/PCSS Projection 詳細 |

---

## ue5-dive 起点

- 「CSM カスケード分割を変えるには」 → `ShadowSetup.cpp:CreateWholeSceneProjectedShadow` + `FShadowCascadeSettings`
- 「Shadow Depth がどこに書かれているか」 → `ShadowRendering.cpp:RenderShadowDepthMaps` + Shadow Atlas RT
- 「PCF と PCSS の切替」 → `TShadowProjectionPS<PCFSamples>` + `r.Shadow.FilterMethod`
- 「Per-Object Shadow の発動条件」 → `ShadowSetup.cpp:CreatePerObjectProjectedShadow`
- 「VSM と従来シャドウの使い分け」 → `GetLightOcclusionType` + `r.Shadow.Virtual.Enable`
