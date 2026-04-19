# GPU GPUScene ソースマップ

- 対象: GPUScene GPU シェーダー（Upload + Instance Culling + DrawCommand 構築）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_gpuscene_gpu_overview]]

Primitive/Instance データの GPU 転送 → インスタンスフラスタムカリング → DrawIndirect 引数生成。
GPU Driven Rendering の核となる基盤で、Nanite/VSM/Lumen など全ての CPU パスから再利用される。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/GPUScene/GPUSceneDataManagement.usf` |
| シェーダー | `Engine/Shaders/Private/InstanceCulling/BuildInstanceDrawCommands.usf` |
| シェーダー | `Engine/Shaders/Private/InstanceCulling/CompactVisibleInstances.usf` |
| CPU | `Renderer/Private/GPUScene.cpp` / `InstanceCulling/InstanceCullingContext.cpp` |

---

## ファイル → シェーダー対応

### Upload

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `GPUScene/GPUSceneDataManagement.usf` | `GPUSceneSetInstancePrimitiveIdCS()` | `FGPUScene::Update()` / `UploadDynamicPrimitiveShaderDataForView()` | PrimitiveSceneData を GPU バッファへ書き込み | [[detail_upload]] |

### Instance Culling

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `InstanceCulling/BuildInstanceDrawCommands.usf` | `InstanceCullBuildInstanceIdBufferCS()` | `FInstanceCullingContext::BuildRenderingCommands()` | フラスタムカリング → InstanceId バッファ | [[detail_instance_culling]] |
| `InstanceCulling/BuildInstanceDrawCommands.usf` | `ClearIndirectArgInstanceCountCS()` | 同上 | Indirect Args カウントリセット | 同 |
| `InstanceCulling/CompactVisibleInstances.usf` | `CompactVisibleInstancesCS()` | 同上 | 可視インスタンスコンパクト化 → DrawIndirect | 同 |

### Build Info

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| GPUScene 内部 CS | `UpdatePrimitivesToRenderCS()` | `FScene::UpdateAllPrimitiveSceneInfos()` | マテリアル・ライトデータ転送 | [[detail_build_info]] |

---

## GPU データフロー

```
[1] Upload                           FGPUScene::Update
    GPUSceneDataManagement.usf:GPUSceneSetInstancePrimitiveIdCS
    → GPUScene.PrimitiveBuffer / InstanceDataBuffer

[2] Instance Culling                 FInstanceCullingContext::BuildRenderingCommands
    BuildInstanceDrawCommands.usf:InstanceCullBuildInstanceIdBufferCS
    BuildInstanceDrawCommands.usf:ClearIndirectArgInstanceCountCS
    → InstanceId バッファ + IndirectArgs

[3] Compact                          同上
    CompactVisibleInstances.usf:CompactVisibleInstancesCS
    → DrawIndirect 用インスタンス ID バッファ

[4] Build Info                       FScene::UpdateAllPrimitiveSceneInfos
    専用 CS → LightData / SceneBuildInfo 転送
```

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_upload]] / [[detail_instance_culling]] / [[detail_build_info]] |
| Reference | [[ref_upload]] / [[ref_instance_culling]] / [[ref_build_info]] |

---

## ue5-dive 起点

- 「GPUScene 転送」 → `GPUSceneDataManagement.usf:GPUSceneSetInstancePrimitiveIdCS`
- 「Instance Culling 本体」 → `BuildInstanceDrawCommands.usf:InstanceCullBuildInstanceIdBufferCS`
- 「Draw Indirect 構築」 → `CompactVisibleInstances.usf:CompactVisibleInstancesCS`
- 「CPU 呼び出し元」 → `FInstanceCullingContext::BuildRenderingCommands()`
