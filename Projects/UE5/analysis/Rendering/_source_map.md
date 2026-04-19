# Rendering ソースマップ

- 対象: UE5 レンダリングサブシステム全体（Renderer / RHI / RenderCore / Shaders）
- 更新日: 2026-04-19
- 上位: [[_module_index]]
- 全体概要: [[01_rendering_overview]]

各サブフォルダ（`BasePass/`, `Lumen/`, `Nanite/` 等）の詳細は別ファイル
`{Subfolder}/_source_map.md` に順次作成。本ファイルはシステム全体の入口。

---

## モジュール構成

| モジュール | ソースパス | ファイル数 | 解析フォルダ | 役割 |
|-----------|----------|----------|------------|------|
| Renderer (Private) | `Engine/Source/Runtime/Renderer/Private/` | 274 h+cpp | `Rendering/` | CPU 側レンダリングパイプライン本体 |
| Renderer (Public) | `Engine/Source/Runtime/Renderer/Public/` | — | `Rendering/` | 公開 API（外部モジュールから参照可） |
| RHI | `Engine/Source/Runtime/RHI/` | — | `Rendering/RHI/` | ハードウェア抽象化レイヤ |
| RenderCore | `Engine/Source/Runtime/RenderCore/` | — | `Rendering/RDG/` | RDG（レンダーグラフ）・共通ユーティリティ |
| Shaders | `Engine/Shaders/Private/` | 489 usf | `Rendering/GPU/` | HLSL シェーダーコード本体 |
| Shaders (Public) | `Engine/Shaders/Public/` | — | `Rendering/GPU/` | 公開インクルード（MaterialGraph から参照） |

---

## トップレベル主要ファイル → クラス対応

レンダラー全体の中心となるファイル群。各サブフォルダに入る前に押さえておくべきコア。

### レンダリングエントリ

| ファイル | 主要クラス / 関数 | 役割 | 参照先 |
|---------|----------------|------|-------|
| `Private/DeferredShadingRenderer.h/cpp` | `FDeferredShadingSceneRenderer` | Deferred Shading のメインレンダラー。`Render()` が全パスを呼ぶ | [[Rendering/Reference/ref_deferred_shading_renderer]] |
| `Private/SceneRendering.h/cpp` | `FSceneRenderer`, `FViewInfo` | レンダラー基底 + レンダースレッド拡張ビュー | [[Rendering/Reference/ref_scene_renderer]], [[Rendering/Reference/ref_view_info]] |
| `Private/SceneRenderBuilder.h/cpp` | `FSceneRenderBuilder` | レンダーコマンド積み上げ・実行 | — |
| `Private/ScenePrivate.h` | `FScene` | レンダースレッド側シーン | [[Rendering/Reference/ref_scene]] |
| `Private/ScenePrivate.cpp` | `FScene::*` | プリミティブ登録・更新 | — |

### シーン構成

| ファイル | 主要クラス | 役割 |
|---------|----------|------|
| `Private/PrimitiveSceneInfo.h/cpp` | `FPrimitiveSceneInfo` | プリミティブのレンダースレッド表現 |
| `Private/PrimitiveSceneProxy.h` | `FPrimitiveSceneProxy` | プリミティブのレンダープロキシ基底 |
| `Private/SceneCore.h/cpp` | `FStaticMeshBatch`, `FLightPrimitiveInteraction` | 静的メッシュバッチ・光源相互作用 |
| `Private/LightSceneInfo.h/cpp` | `FLightSceneInfo` | ライトのシーン情報 |

### レンダリングパイプライン中核

| ファイル | 主要クラス/関数 | 役割 | サブフォルダ |
|---------|--------------|------|----------|
| `Private/DepthRendering.h/cpp` | `FDepthPassMeshProcessor` | PrePass（深度先行描画） | [[DepthPrepass/]] |
| `Private/BasePassRendering.h/cpp` | `FBasePassMeshProcessor` | GBuffer パス | [[BasePass/]] |
| `Private/LightRendering.cpp` | `FDeferredShadingSceneRenderer::RenderLights()` | 直接光ライティング | [[DeferredLighting/]] |
| `Private/ShadowRendering.h/cpp` | `FProjectedShadowInfo` | シャドウ投影 | [[Shadow/]] |
| `Private/TranslucentRendering.cpp` | `FTranslucencyPassMeshProcessor` | 半透明パス | [[Translucency/]] |
| `Private/DistanceFieldAmbientOcclusion.cpp` | — | SSAO / DFAO | [[SSAO/]], [[DistanceField/]] |
| `Private/FogRendering.cpp` | `FExponentialHeightFogSceneInfo` | Height Fog / Volumetric Fog | [[Fog/]] |
| `Private/AtmosphereRendering.cpp` | — | Sky Atmosphere | [[SkyAtmosphere/]] |
| `Private/DecalRenderingShared.h/cpp` | `FDecalRenderingCommon` | Deferred デカール | [[Decals/]] |

### GPU Driven / 次世代機能

| ファイル | 主要クラス | 役割 | サブフォルダ |
|---------|----------|------|----------|
| `Private/GPUScene.h/cpp` | `FGPUScene`, `FGPUSceneWriter` | GPU 側シーン全インスタンスデータ | [[GPUScene/]] |
| `Private/Nanite/NaniteShared.h` | `Nanite::FPackedView`, `Nanite::FPackedViewArray` | Nanite 共通データ | [[Nanite/]] |
| `Private/Lumen/LumenSceneData.h` | `FLumenSceneData`, `FLumenCardScene` | Lumen シーン | [[Lumen/]] |
| `Private/VirtualShadowMaps/VirtualShadowMapArray.h/cpp` | `FVirtualShadowMapArray` | VSM 全体管理 | [[VirtualShadowMaps/]] |
| `Private/MegaLights/MegaLights.h` | `FMegaLightsFrameTemporaries` | 多数ライトの GPU Driven 処理 | [[MegaLights/]] |
| `Private/RayTracing/RayTracingScene.h` | `FRayTracingScene` | レイトレーシングシーン | [[RayTracing/]] |
| `Private/Substrate/Substrate.h` | — | 新マテリアルレイヤー（UE5.3〜） | [[Substrate/]] |
| `Private/HZB.h/cpp` | `FHZBOcclusionTester` | 階層化 Z バッファ | [[HZB/]] |

### カリング・可視性

| ファイル | 主要クラス | 役割 |
|---------|----------|------|
| `Private/SceneVisibility.cpp` | `FSceneRenderer::InitViews()` 群 | 視錐台カリング・オクルージョン |
| `Private/MeshDrawCommands.h/cpp` | `FMeshDrawCommand`, `FMeshPassProcessor` | メッシュ描画コマンド管理 |
| `Private/MeshPassProcessor.h/cpp` | `FMeshPassProcessor` | パス別メッシュプロセッサ基底 |

### ポストプロセス

| ファイル | 主要クラス | 役割 | サブフォルダ |
|---------|----------|------|----------|
| `Private/PostProcess/PostProcessing.h/cpp` | `AddPostProcessingPasses()` | PP パスディスパッチ | [[PostProcess/]] |
| `Private/PostProcess/TemporalAA.cpp` | — | TAA / TSR | [[PostProcess/]] |
| `Private/PostProcess/PostProcessTonemap.cpp` | — | Tonemap | [[PostProcess/]] |
| `Public/TemporalUpscaler.h` | `ITemporalUpscaler` | TSR 公開 API | [[PostProcess/]] |

### RHI / RDG

| ファイル | 主要クラス | 役割 | サブフォルダ |
|---------|----------|------|----------|
| `RenderCore/Public/RenderGraphBuilder.h` | `FRDGBuilder` | Render Graph 組立 | [[RDG/]] |
| `RenderCore/Public/RenderGraphResources.h` | `FRDGTexture`, `FRDGBuffer` | RDG リソース型 | [[RDG/]] |
| `RHI/Public/RHICommandList.h` | `FRHICommandListImmediate` | RHI コマンドリスト | [[RHI/]] |
| `RHI/Public/RHI.h` | `FRHITexture`, `FRHIBuffer` | RHI リソース型 | [[RHI/]] |
| `RHI/Public/DynamicRHI.h` | `FDynamicRHI` | RHI バックエンド基底 | [[RHI/]] |

---

## サブフォルダ一覧（CPU 側 23）

解析済みの CPU 側サブシステム。各フォルダに `01_overview.md` / `Details/` / `Reference/` が揃っている。
`_source_map.md` は on-demand で作成予定（`[ ]` は未作成）。

| サブフォルダ | ソースパス（主要） | 概要リンク | ソースマップ |
|------------|----------------|----------|-----------|
| `BasePass/` | `Private/BasePassRendering.*` | [[BasePass/01_overview]] | [ ] |
| `Decals/` | `Private/DecalRenderingShared.*` | [[Decals/01_overview]] | [ ] |
| `DeferredLighting/` | `Private/LightRendering.cpp` | [[DeferredLighting/01_overview]] | [ ] |
| `DepthPrepass/` | `Private/DepthRendering.*` | [[DepthPrepass/01_overview]] | [ ] |
| `DistanceField/` | `Private/DistanceField*.cpp` | [[DistanceField/01_overview]] | [ ] |
| `Fog/` | `Private/FogRendering.cpp`, `VolumetricFog.cpp` | [[Fog/01_overview]] | [ ] |
| `GPUScene/` | `Private/GPUScene.*` | [[GPUScene/01_overview]] | [ ] |
| `HZB/` | `Private/HZB.*` | [[HZB/01_overview]] | [ ] |
| `Lumen/` | `Private/Lumen/` | [[Lumen/02_lumen_overview]] | [ ] |
| `MegaLights/` | `Private/MegaLights/` | [[MegaLights/06_megalights_overview]] | [ ] |
| `MeshPassProcessor/` | `Private/MeshPassProcessor.*`, `MeshDrawCommands.*` | [[MeshPassProcessor/11_mpp_overview]] | [ ] |
| `Nanite/` | `Private/Nanite/` | [[Nanite/03_nanite_overview]] | [ ] |
| `PostProcess/` | `Private/PostProcess/` | [[PostProcess/05_postprocess_overview]] | [ ] |
| `RayTracing/` | `Private/RayTracing/` | [[RayTracing/07_raytracing_overview]] | [ ] |
| `RDG/` | `RenderCore/Public/RenderGraph*.h`, `RenderCore/Private/` | [[RDG/10_rdg_overview]] | [ ] |
| `RHI/` | `RHI/Public/`, `RHI/Private/` | [[RHI/12_rhi_overview]] | [ ] |
| `SceneRenderer/` | `Private/SceneRendering.*`, `DeferredShadingRenderer.*` | [[SceneRenderer/01_overview]] | [ ] |
| `Shadow/` | `Private/ShadowRendering.*`, `ShadowSetup.cpp` | [[Shadow/01_overview]] | [ ] |
| `SkyAtmosphere/` | `Private/AtmosphereRendering.cpp`, `SkyAtmosphereRendering.*` | [[SkyAtmosphere/01_overview]] | [ ] |
| `SSAO/` | `Private/CompositionLighting/` 内 SSAO 関連 | [[SSAO/01_overview]] | [ ] |
| `Substrate/` | `Private/Substrate/` | [[Substrate/08_substrate_overview]] | [ ] |
| `Translucency/` | `Private/TranslucentRendering.*` | [[Translucency/01_overview]] | [ ] |
| `VirtualShadowMaps/` | `Private/VirtualShadowMaps/` | [[VirtualShadowMaps/04_vsm_overview]] | [ ] |

---

## サブフォルダ一覧（GPU / Shaders 側 17）

`Engine/Shaders/Private/` 以下の HLSL シェーダー群。CPU 側と対になっている機能ごとに分類。

| サブフォルダ | ソースパス（主要 .usf） | 概要リンク | ソースマップ |
|------------|---------------------|----------|-----------|
| `GPU/BasePass/` | `Shaders/Private/BasePassPixelShader.usf`, `BasePassVertexShader.usf` | [[GPU/BasePass/01_overview]] | [ ] |
| `GPU/Decals/` | `Shaders/Private/DeferredDecal.usf` | [[GPU/Decals/01_overview]] | [ ] |
| `GPU/DeferredLighting/` | `Shaders/Private/DeferredLightPixelShaders.usf` | [[GPU/DeferredLighting/01_overview]] | [ ] |
| `GPU/DepthPrepass/` | `Shaders/Private/DepthOnlyVertexShader.usf`, `DepthOnlyPixelShader.usf` | [[GPU/DepthPrepass/01_overview]] | [ ] |
| `GPU/DistanceField/` | `Shaders/Private/DistanceField*.usf` | [[GPU/DistanceField/01_overview]] | [ ] |
| `GPU/Fog/` | `Shaders/Private/HeightFogPixelShader.usf`, `VolumetricFog*.usf` | [[GPU/Fog/01_overview]] | [ ] |
| `GPU/GPUScene/` | `Shaders/Private/GPUScene*.ush` | [[GPU/GPUScene/01_overview]] | [ ] |
| `GPU/HZB/` | `Shaders/Private/HZB*.usf` | [[GPU/HZB/01_overview]] | [ ] |
| `GPU/Lumen/` | `Shaders/Private/Lumen/*.usf` | [[GPU/Lumen/01_overview]] | [ ] |
| `GPU/MegaLights/` | `Shaders/Private/MegaLights/*.usf` | [[GPU/MegaLights/01_overview]] | [ ] |
| `GPU/Nanite/` | `Shaders/Private/Nanite/*.usf` | [[GPU/Nanite/01_overview]] | [ ] |
| `GPU/PostProcess/` | `Shaders/Private/PostProcess*.usf`, `TSR/*.usf` | [[GPU/PostProcess/01_overview]] | [ ] |
| `GPU/RayTracing/` | `Shaders/Private/RayTracing/*.usf` | [[GPU/RayTracing/01_overview]] | [ ] |
| `GPU/SkyAtmosphere/` | `Shaders/Private/SkyAtmosphere.usf` | [[GPU/SkyAtmosphere/01_overview]] | [ ] |
| `GPU/SSAO/` | `Shaders/Private/PostProcessAmbientOcclusion.usf` | [[GPU/SSAO/01_overview]] | [ ] |
| `GPU/Translucency/` | `Shaders/Private/TranslucentLightingShaders.usf` | [[GPU/Translucency/01_overview]] | [ ] |
| `GPU/VirtualShadowMaps/` | `Shaders/Private/VirtualShadowMaps/*.usf` | [[GPU/VirtualShadowMaps/01_overview]] | [ ] |

> Substrate / SceneRenderer / MeshPassProcessor は GPU 側フォルダを持たず、各機能の .usf が他フォルダに散在。

---

## レンダリングパイプライン俯瞰（ソースファイル対応）

`FDeferredShadingSceneRenderer::Render()` 内の主要パス順と対応ソース。

```
Frame 開始
  │
  ├─ InitViews                        SceneVisibility.cpp
  │     └─ FrustumCull / HZB / Relevance
  │
  ├─ GPUScene.Update                  GPUScene.cpp               [[GPUScene/]]
  │
  ├─ RenderPrePass                    DepthRendering.cpp         [[DepthPrepass/]]
  │
  ├─ Nanite::CullRasterize            Nanite/NaniteCullRaster.cpp [[Nanite/]]
  │     └─ VisBuffer64
  │
  ├─ VirtualShadowMapArray.Render     VirtualShadowMaps/         [[VirtualShadowMaps/]]
  │
  ├─ RenderBasePass                   BasePassRendering.cpp      [[BasePass/]]
  │     ├─ Nanite::DrawMaterialShading
  │     └─ 非 Nanite → GBuffer
  │
  ├─ RenderDecals                     DecalRenderingShared.cpp   [[Decals/]]
  │
  ├─ LumenScene.Update                Lumen/LumenScene.cpp       [[Lumen/]]
  │
  ├─ RenderDiffuseIndirectAndAO       Lumen/LumenDiffuseIndirect + DFAO [[Lumen/]], [[DistanceField/]]
  │
  ├─ RenderShadows                    ShadowRendering.cpp        [[Shadow/]]
  │
  ├─ RenderLights                     LightRendering.cpp         [[DeferredLighting/]]
  │
  ├─ MegaLights::Render               MegaLights/                [[MegaLights/]]
  │
  ├─ RenderReflections + SkyLight     LumenReflections.cpp       [[Lumen/]], [[SkyAtmosphere/]]
  │
  ├─ RenderFog                        FogRendering.cpp           [[Fog/]]
  │
  ├─ RenderTranslucency               TranslucentRendering.cpp   [[Translucency/]]
  │
  └─ AddPostProcessingPasses          PostProcess/PostProcessing.cpp [[PostProcess/]]
```

---

## ue5-dive での使い方

調査したい機能が決まったら：

1. **このマップで該当サブフォルダを特定**（例: Nanite のカリング → `Nanite/` + `Nanite/NaniteCullRaster.cpp`）
2. **`{Subfolder}/_source_map.md` があれば参照**（なければ on-demand で作成）
3. **`{Subfolder}/Reference/*` で詳細 API を確認**
4. **`{Subfolder}/Details/*` で実装ロジックを確認**
5. 実ソースを読む必要があれば `D:\UnrealEngine\...` を開く

---

## 未作成ソースマップ

システム全体マップ以外のサブフォルダ別 `_source_map.md` は未作成。
優先度は [[TASK_CHECKLIST_SOURCE_MAP]] を参照。頻出は Lumen / Nanite / VSM / RDG / RHI。
