# GPUScene ソースマップ

- 対象: GPU 駆動レンダリングの基盤バッファ（`FGPUScene` 中心）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[09_gpuscene_overview]]

シーン全プリミティブ・インスタンス・ライト・ライトマップを永続 GPU バッファに保持し、
Nanite / Lumen / VSM / RayTracing の共通データソースとなる。

---

## ソースパス

| 対象 | パス |
|------|------|
| GPUScene 本体 | `Engine/Source/Runtime/Renderer/Private/GPUScene.h/.cpp` |
| シーン側呼び出し元 | `Engine/Source/Runtime/Renderer/Private/RendererScene.cpp` |
| BeginRender 呼び出し元 | `Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp` |
| Scatter Upload CS | `Engine/Shaders/Private/GPUScene/GPUSceneDataUpload.usf` |

---

## ファイル → クラス対応

### 本体

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `Private/GPUScene.h` | `FGPUScene`, `FGPUSceneResourceParameters`, `FGPUScenePrimitiveCollector`, `FGPUSceneScopeBeginEndHelper`:513 | シーンバッファ本体・シェーダーパラメータ・動的プリミティブ収集 | [[Reference/ref_gpuscene_class]], [[Reference/ref_gpuscene_params]] |
| `Private/GPUScene.cpp` | `BeginRender`:726, `EndRender`:746, `UpdateGPULights`:765, `UpdateInternal`:883, `Update`:1978, `UploadDynamicPrimitiveShaderDataForView`:1993 | Scatter Upload・並列更新 | [[Details/a_gpuscene_buffers]], [[Details/b_gpuscene_update]], [[Details/c_gpuscene_dynamic]] |

### 関連（アロケータ・登録元）

| ファイル | 主要クラス | 役割 |
|---------|----------|------|
| `Private/SpanAllocator.h` | `FSpanAllocator` | インスタンスデータスロット割り当て |
| `Private/RendererScene.cpp` | `FScene::UpdateAllPrimitives`, `FGPUScene::Update`:6541 からの呼び出し | シーン更新時のアップロード起動 |
| `Private/PrimitiveSceneInfo.cpp` | `FPrimitiveSceneInfo::MarkPrecomputedLightingBufferDirty` など | プリミティブ変化の通知 |

### シェーダー側（GPU）

| ファイル | 役割 | サブフォルダ |
|---------|------|-----------|
| `Shaders/Private/GPUScene/GPUSceneDataUpload.usf` | Scatter Upload CS | [[GPU/GPUScene/]] |
| `Shaders/Private/SceneData.ush` | プリミティブ/インスタンス読み出しヘルパー | [[GPU/GPUScene/]] |

---

## バッファ構成（FGPUSceneResourceParameters）

シェーダーから参照される SRV 一覧。`GPUScene.h:FGPUSceneResourceParameters` に定義。

| バッファ | 型 | 内容 |
|---------|----|-----|
| `GPUScenePrimitiveSceneData` | `StructuredBuffer<float4>` | プリミティブ変換行列・バウンド |
| `GPUSceneInstanceSceneData` | `StructuredBuffer<float4>` | インスタンス ローカル→ワールド |
| `GPUSceneInstancePayloadData` | `StructuredBuffer<float4>` | カスタムペイロード |
| `GPUSceneLightmapData` | `StructuredBuffer<float4>` | ライトマップ UV |
| `GPUSceneLightData` | `ByteAddressBuffer` | ライトデータ |

共通パラメータ:
- `GPUSceneInstanceDataSOAStride`
- `GPUSceneFrameNumber`
- `GPUSceneMaxAllocatedInstanceId`
- `GPUSceneMaxPersistentPrimitiveIndex`

---

## フレームライフサイクル → ソース位置

```
FDeferredShadingSceneRenderer::Render()                          DeferredShadingRenderer.cpp
  │
  ├─[A] FGPUSceneScopeBeginEndHelper ctor                        :1799
  │      └─ FGPUScene::BeginRender                               GPUScene.cpp:726
  │           ├─ DynamicPrimitivesOffset リセット
  │           └─ RegisterBuffers → CachedRegisteredBuffers
  │
  ├─[B] FScene::UpdateAllPrimitives                              RendererScene.cpp
  │      └─ FGPUScene::Update                                    RendererScene.cpp:6541
  │           └─ UpdateInternal                                  GPUScene.cpp:883
  │                ├─ PrimitivesToUpdate イテレート
  │                ├─ ScatterUpload バッファ書き込み
  │                └─ FillSceneUniformBuffer (SceneUB 更新)
  │
  ├─[C] UpdateGPULights                                          GPUScene.cpp:765
  │      └─ ライト差分 ByteAddressBuffer へ upload
  │
  ├─[D] UploadDynamicPrimitiveShaderDataForView                  DeferredShadingRenderer.cpp:2172
  │      └─ View.DynamicPrimitiveCollector → GPU 動的領域         GPUScene.cpp:1993
  │
  └─[E] FGPUSceneScopeBeginEndHelper dtor
         └─ FGPUScene::EndRender                                 GPUScene.cpp:746
```

---

## 主要 CVar

| CVar | 対応位置 |
|------|--------|
| `r.GPUScene.UploadEveryFrame` | `GPUScene.cpp` — デバッグ強制アップロード |
| `r.GPUScene.ParallelUpdate` | `GPUScene.cpp` — 並列タスクで更新 |
| `r.GPUScene.MaxPooledUploadBufferSize` | `GPUScene.cpp` — アップロードバッファプール上限 |

---

## 参照するサブシステム

GPUScene バッファは以下から読み出される。各サブフォルダ参照。

| サブシステム | 典型アクセス | サブフォルダ |
|------------|------------|-----------|
| Nanite | `GetSceneData()` in `NaniteShared.ush` | [[Nanite/]] |
| Lumen | `LumenCardPass` / `LumenRadianceCache` | [[Lumen/]] |
| VSM | `VirtualShadowMapProjection` | [[VirtualShadowMaps/]] |
| RayTracing | `FRayTracingScene::BuildInstances` | [[RayTracing/]] |
| 通常メッシュ | `FMeshDrawCommand` が SceneUB 経由で参照 | [[MeshPassProcessor/]] |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_gpuscene_class]] | FGPUScene クラス |
| Reference | [[Reference/ref_gpuscene_params]] | FGPUSceneResourceParameters |
| Details | [[Details/a_gpuscene_buffers]] | バッファ構成・SOA レイアウト |
| Details | [[Details/b_gpuscene_update]] | Scatter Upload 並列化 |
| Details | [[Details/c_gpuscene_dynamic]] | 動的プリミティブ領域 |

---

## ue5-dive 起点

- 「プリミティブ追加が GPU に届くまで」 → `RendererScene.cpp:FScene::AddPrimitive` → `GPUScene.cpp:UpdateInternal:883`
- 「動的プリミティブの仕組み」 → `FGPUScenePrimitiveCollector` + `UploadDynamicPrimitiveShaderDataForView:1993`
- 「シェーダーからプリミティブ読み出し」 → `Shaders/Private/SceneData.ush:GetPrimitiveData`
- 「Scatter Upload の実行」 → `GPUScene.cpp:UpdateInternal` + `GPUSceneDataUpload.usf`
