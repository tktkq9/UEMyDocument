# GPUScene GPU 処理概要

- グループ: GPUScene GPU
- CPU 概要: [[01_gpuscene_overview]]
- CPU 詳細: [[a_gpuscene_upload]] | [[b_instance_culling]] | [[c_gpuscene_build]]

---

## GPUScene GPU パイプライン実行順

GPUScene は **PrimitiveSceneData / InstanceSceneData** の GPU 常駐バッファを管理し、  
CPU→GPU 転送、インスタンスカリング、DrawCommand 生成を行う。

```
[Upload（CPU→GPU 転送）]
[1]  Primitive / Instance データ転送
     └─ GPUSceneSetInstancePrimitiveIdCS（GPUSceneDataManagement.usf）
          CPU が更新した PrimitiveSceneData を GPU バッファに書き込み

[Instance Culling（GPU Driven Culling）]
[2]  Instance Cull + DrawCommand 構築
     ├─ InstanceCullBuildInstanceIdBufferCS（BuildInstanceDrawCommands.usf）
     │   各インスタンスをフラスタムカリング → InstanceId バッファ生成
     └─ ClearIndirectArgInstanceCountCS（BuildInstanceDrawCommands.usf）
          Indirect Draw 引数のカウントをリセット

[3]  Compact Visible Instances
     └─ CompactVisibleInstancesCS（CompactVisibleInstances.usf）
          可視インスタンスをコンパクト化 → DrawIndirect 引数バッファ生成

[BuildInfo（シーン情報更新）]
[4]  SceneInfo / LightData 転送
     └─ UpdatePrimitivesToRenderCS（GPUScene 内部 CS）
          マテリアル・ライトデータを GPU に転送
```

---

## 各ステップの詳細

### [1] Upload

| 項目 | 内容 |
|-----|------|
| **概要** | 変更された Primitive / Instance の SceneData を GPU バッファに書き込む |
| **CPU 関数** | `FGPUScene::Update()` / `UploadDynamicPrimitiveShaderDataForView()` |
| **シェーダー** | CS: `GPUSceneDataManagement.usf#GPUSceneSetInstancePrimitiveIdCS()` |
| **出力** | `GPUScene.PrimitiveBuffer` / `GPUScene.InstanceDataBuffer` |
| **GPU シェーダー詳細** | [[detail_upload]] / [[ref_upload]] |

---

### [2〜3] Instance Culling

| 項目 | 内容 |
|-----|------|
| **概要** | GPU Driven Rendering の核。インスタンスをフラスタムカリングして DrawIndirect 用引数を構築 |
| **CPU 関数** | `FInstanceCullingContext::BuildRenderingCommands()` |
| **シェーダー** | `BuildInstanceDrawCommands.usf#InstanceCullBuildInstanceIdBufferCS()` / `CompactVisibleInstances.usf` |
| **出力** | `DrawIndirect` 用インスタンス ID バッファ |
| **GPU シェーダー詳細** | [[detail_instance_culling]] / [[ref_instance_culling]] |

---

### [4] BuildInfo

| 項目 | 内容 |
|-----|------|
| **概要** | LightData / SceneBuildInfo を GPU に転送 |
| **CPU 関数** | `FScene::UpdateAllPrimitiveSceneInfos()` → GPU バッファ更新 |
| **シェーダー** | 専用 CS（GPUScene 内部）|
| **GPU シェーダー詳細** | [[detail_build_info]] / [[ref_build_info]] |

---

## シェーダー別 CPU 対応一覧

| シェーダーファイル | 主要エントリポイント | CPU 関数 | グループ |
|----------------|----------------|---------|---------|
| `GPUScene/GPUSceneDataManagement.usf` | `GPUSceneSetInstancePrimitiveIdCS()` | `FGPUScene::Update()` | [[a_Upload]] |
| `InstanceCulling/BuildInstanceDrawCommands.usf` | `InstanceCullBuildInstanceIdBufferCS()` | `FInstanceCullingContext::BuildRenderingCommands()` | [[b_InstanceCulling]] |
| `InstanceCulling/BuildInstanceDrawCommands.usf` | `ClearIndirectArgInstanceCountCS()` | `FInstanceCullingContext::BuildRenderingCommands()` | [[b_InstanceCulling]] |
| `InstanceCulling/CompactVisibleInstances.usf` | `CompactVisibleInstancesCS()` | `FInstanceCullingContext::BuildRenderingCommands()` | [[b_InstanceCulling]] |
