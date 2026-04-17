# Instance Culling シェーダー詳細

- グループ: b - InstanceCulling
- GPU 概要: [[01_gpuscene_gpu_overview]]
- CPU 詳細: [[b_instance_culling]]
- リファレンス: [[ref_instance_culling]]

---

## 概要

**GPU Driven Instance Culling** はインスタンスをフラスタムカリングして  
`DrawIndexedPrimitiveIndirect` 用のインスタンス ID バッファを GPU 上で構築する。  
CPU での可視性計算を GPU に移譲することで、大量のインスタンスを効率的に処理できる。

---

## レンダリングパスの構成

```
Instance Culling パス
  │
  ├─ [ClearIndirectArgs]
  │   CS: ClearIndirectArgInstanceCountCS()
  │   DrawIndirect 引数のインスタンスカウントを 0 にリセット
  │
  ├─ [Instance Cull + DrawCommand 構築]
  │   CS: InstanceCullBuildInstanceIdBufferCS()
  │   各インスタンスを:
  │     1. フラスタムカリング（AABB vs View Frustum）
  │     2. HZB Occlusion カリング（有効時）
  │     3. 通過したインスタンスを InstanceId バッファに書き込み
  │     4. DrawIndirect 引数のカウントを AtomicAdd で更新
  │
  └─ [Compact（オプション）]
      CS: CompactVisibleInstancesCS()
      スパースな InstanceId バッファをコンパクト化
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `GPUScene.InstanceDataBuffer` | 全インスタンスの PrimitiveId / Bounds / Flags |
| `GPUScene.PrimitiveBuffer` | Primitive の変換行列・Bounds |
| `FrustumPlanes` | ビュー フラスタム平面（6 平面）|
| `HZBTexture`（オプション）| FurthestHZB ミップチェーン |

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| `InstanceIdBuffer` | `RWBuffer<uint>` | 可視インスタンスの InstanceId 列 |
| `DrawIndirectArgsBuffer` | `RWBuffer<uint4>` | DrawIndexedPrimitiveIndirect の引数 |

---

## シェーダーコアロジック

### ClearIndirectArgInstanceCountCS（:363）

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void ClearIndirectArgInstanceCountCS(uint3 GroupId : SV_GroupID, int GroupThreadIndex : SV_GroupIndex)
{
    // DrawIndirect 引数バッファのインスタンスカウントを 0 にクリア
    FInstanceWorkSetup WorkSetup = InstanceCullingLoadBalancer_Setup(...);
    if (!WorkSetup.bValid) return;

    uint DrawCommandId = WorkSetup.Item.Payload;
    // RWDrawIndirectArgsBuffer[DrawCommandId].InstanceCount = 0;
    ClearIndirectArgInstanceCount(DrawCommandId);
}
```

### InstanceCullBuildInstanceIdBufferCS（:235）

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void InstanceCullBuildInstanceIdBufferCS(uint3 GroupId : SV_GroupID, int GroupThreadIndex : SV_GroupIndex)
{
    FInstanceWorkSetup WorkSetup = InstanceCullingLoadBalancer_Setup(...);
    if (!WorkSetup.bValid) return;

    uint InstanceId = WorkSetup.Item.InstanceDataOffset + uint(WorkSetup.LocalItemIndex);

    // GPUScene から Instance データを読み込み
    FInstanceSceneData InstanceData = GetInstanceSceneData(InstanceId, false);
    FPrimitiveSceneData PrimitiveData = GetPrimitiveData(InstanceData.PrimitiveId);

    // 非表示フラグチェック
    if (InstanceData.Flags & INSTANCE_SCENE_DATA_FLAG_HIDDEN) return;

    // フラスタムカリング
    FFrustumCullData FrustumCull = FrustumCullSphere(
        PrimitiveData.LocalToWorld,
        PrimitiveData.ObjectBounds,
        FrustumPlanes);

    if (!FrustumCull.bIsVisible) return;

    // HZB Occlusion Test（オプション）
    if (bHZBOcclusion && !HZBOcclusionTest(PrimitiveData, FrustumCull)) return;

    // 可視インスタンスを InstanceId バッファに追加
    uint WriteOffset;
    InterlockedAdd(RWDrawIndirectArgsBuffer[DrawCommandId].InstanceCount, 1, WriteOffset);
    RWInstanceIdBuffer[BaseOffset + WriteOffset] = InstanceId;
}
```

---

## CPU 呼び出しの流れ

```
FInstanceCullingContext::BuildRenderingCommands()    // InstanceCullingContext.cpp
  │
  ├─ LoadBalancer にインスタンスバッチを積む
  ├─ AddPass(ClearIndirectArgInstanceCountCS)
  ├─ AddPass(InstanceCullBuildInstanceIdBufferCS)
  │   GroupCount = LoadBalancer.GetNumBatches()
  │
  └─ InstanceIdBuffer + DrawIndirectArgsBuffer を各パスにバインド
      → FMeshDrawCommand::Execute() で DrawIndexedPrimitiveIndirect 呼び出し
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `InstanceCulling/BuildInstanceDrawCommands.usf` | CS | フラスタムカリング + DrawIndirect 引数構築 |
| `InstanceCulling/CompactVisibleInstances.usf` | CS | 可視インスタンスのコンパクト化 |
| `InstanceCulling/InstanceCullingLoadBalancer.ush` | ヘッダ | ワークアイテム分散（グループ ID → InstanceId 変換）|
| `SceneData.ush` | ヘッダ | `GetInstanceSceneData()` / `GetPrimitiveData()` アクセサ |
