# BasePass ソースマップ

- 対象: BasePass（GBuffer 書き込み / マテリアル評価）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[14_basepass_overview]]

Deferred Rendering の GBuffer 書き込みパス。`TBasePassMeshProcessor` が
マテリアル × ライトマップ種別でシェーダー順列を選び、GBufferA〜F + SceneColor + Velocity を出力する。
Nanite メッシュは VisBuffer → `NaniteShading::EmitGBuffer` 経由で別ルートを通る。

---

## ソースパス

| 対象 | パス |
|------|------|
| BasePass 本体 | `Engine/Source/Runtime/Renderer/Private/BasePassRendering.h/.cpp` |
| テンプレートシェーダー | `Engine/Source/Runtime/Renderer/Private/BasePassRendering.inl` |
| SceneTextures（GBuffer 束） | `Engine/Source/Runtime/Renderer/Private/SceneTextures.h/.cpp` |
| GBuffer デコード | `Engine/Source/Runtime/Renderer/Public/GBufferInfo.h` |
| Nanite 統合 | `Engine/Source/Runtime/Renderer/Private/Nanite/NaniteShading.h/.cpp` |
| GPU シェーダー | `Engine/Shaders/Private/BasePassPixelShader.usf`, `BasePassVertexShader.usf`, `BasePassCommon.ush` |

---

## ファイル → クラス対応

### BasePass 本体

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `BasePassRendering.h` | `TBasePassMeshProcessor`, `TBasePassVS<>`, `TBasePassPS<>`, `FOpaqueBasePassUniformParameters` | テンプレートベースのプロセッサ・シェーダー・UB | [[Reference/ref_basepass_renderer]], [[Reference/ref_basepass_common]] |
| `BasePassRendering.cpp` | `RenderBasePass()`:1071, `RenderBasePassInternal()`:1449, `RenderNaniteBasePass()`:1516, `CreateOpaqueBasePassUniformBuffer()` | RDG 登録・並列描画・Nanite 分岐 | [[Details/a_basepass_pipeline]], [[Details/c_basepass_material]] |
| `BasePassRendering.inl` | `TBasePassPS<LightMapPolicy, NumMovablePointLights>` | マテリアル × ライトマップ種別の順列実装 | 同 |

### GBuffer 管理

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `SceneTextures.h/.cpp` | `FSceneTextures::GetGBufferRenderTargets()` | GBuffer テクスチャ束のバインド | [[Reference/ref_gbuffer_textures]] |
| `GBufferInfo.h` | `FGBufferData`, `EGBufferSlot`, `EncodeGBuffer() / DecodeGBuffer()` | GBuffer レイアウト定義・デコード | [[Details/b_gbuffer_layout]] |

### Nanite 統合

| ファイル | 主要クラス / 関数 | 役割 |
|---------|----------------|------|
| `Nanite/NaniteShading.h` | `BuildShadingCommands()`, `DispatchBasePass()` | ビン別 IndirectDispatch 引数構築 |
| `Nanite/NaniteShading.cpp` | `ShadeBinning()`:1443, `DispatchBasePass()`:1178, `EmitGBuffer` | VisBuffer → GBuffer CS ディスパッチ |

---

## GBuffer レイアウト（従来）

| スロット | フォーマット | 内容 |
|--------|------------|------|
| GBufferA | R8G8B8A8 | WorldNormal + PerObjectData |
| GBufferB | R8G8B8A8 | Metallic / Specular / Roughness / ShadingModelID |
| GBufferC | R8G8B8A8 | BaseColor + IndirectIrradiance/AO |
| GBufferD | R8G8B8A8 | CustomData（ShadingModel 依存）|
| GBufferE | R8G8B8A8 | Pre-computed shadow factor |
| GBufferF | R8G8B8A8 | WorldTangent + Anisotropy |
| Velocity | R16G16 | Motion Vector |
| SceneDepth | D24S8 | Depth + Stencil |

Substrate 有効時は `MaterialTextureArray`（Texture2DArray uint, `r.Substrate.BytesPerPixel` スライス）に置換。

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()
  │
  └─ RenderBasePass()                                 BasePassRendering.cpp:1071
      │
      ├─[1] GBuffer RT 束バインド                     SceneTextures.cpp
      │      FSceneTextures::GetGBufferRenderTargets
      │
      ├─[2] UniformBuffer 構築                        BasePassRendering.cpp
      │      CreateOpaqueBasePassUniformBuffer
      │      └─ ForwardLight / Reflection / Fog / DBuffer / EyeAdaptation
      │
      ├─[3] Nanite EmitGBuffer                        Nanite/NaniteShading.cpp:1178
      │      DispatchBasePass → ShadeBinning → ビン別 CS
      │
      └─[4] RenderBasePassInternal                    BasePassRendering.cpp:1449
             ├─ [r.ParallelBasePass=1] ParallelCommandList
             └─ DrawDynamicMeshPass
                  └─ TBasePassMeshProcessor::AddMeshBatch
                       └─ GetUniformLightMapPolicy → TBasePassPS<Policy,N>
                       └─ BuildMeshDrawCommands → Submit
```

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.ParallelBasePass` | `BasePassRendering.cpp` |
| `r.SelectiveBasePassOutputs` | `BasePassRendering.cpp` |
| `r.ClearGBufferDBeforeBasePass` | `BasePassRendering.cpp` |
| `r.ClearSceneMethod` | `SceneTextures.cpp` |
| `r.AllowGlobalClipPlane` | `BasePassRendering.cpp` |
| `r.BasePassOutputsVelocity` | `BasePassRendering.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_basepass_renderer]] | RenderBasePass エントリ |
| Reference | [[Reference/ref_basepass_common]] | UniformBuffer・共通パラメータ |
| Reference | [[Reference/ref_gbuffer_textures]] | FSceneTextures GBuffer 部 |
| Details | [[Details/a_basepass_pipeline]] | RDG Pass 組立パイプライン |
| Details | [[Details/b_gbuffer_layout]] | GBuffer レイアウト・エンコード |
| Details | [[Details/c_basepass_material]] | マテリアル × ライトマップ順列 |

---

## ue5-dive 起点

- 「GBuffer 書き込みはどこ」 → `BasePassRendering.cpp:RenderBasePass`
- 「Nanite が GBuffer に書く経路」 → `NaniteShading.cpp:DispatchBasePass` → `ShadeBinning`
- 「マテリアル順列の実体」 → `BasePassRendering.inl:TBasePassPS<Policy, NumLights>`
- 「Substrate で GBuffer レイアウトが変わる条件」 → `GBufferInfo.h` + `r.Substrate.BytesPerPixel`
- 「DBuffer デカールは BasePass のどこで読む」 → `CreateOpaqueBasePassUniformBuffer`
