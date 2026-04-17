# REF: Instance Culling シェーダー

- グループ: b - InstanceCulling
- 詳細: [[detail_instance_culling]]
- CPU リファレンス: [[ref_gpuscene_instance_culling]]
- ソース: `Engine/Shaders/Private/InstanceCulling/BuildInstanceDrawCommands.usf`  
          `Engine/Shaders/Private/InstanceCulling/CompactVisibleInstances.usf`

---

## InstanceCullBuildInstanceIdBufferCS（BuildInstanceDrawCommands.usf:235）

### エントリポイント

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void InstanceCullBuildInstanceIdBufferCS(
    uint3 GroupId          : SV_GroupID,
    int   GroupThreadIndex : SV_GroupIndex)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Compute Shader |
| **スレッドグループ** | NUM_THREADS_PER_GROUP = 64（1D）|
| **目的** | フラスタムカリング → DrawIndirect 引数 InstanceCount AtomicAdd |

#### 出力バインド

| バインド名 | 型 | 説明 |
|-----------|------|------|
| `RWInstanceIdBuffer` | `RWBuffer<uint>` | 可視インスタンスの ID 列 |
| `RWDrawIndirectArgsBuffer` | `RWBuffer<uint4>` | DrawIndexedPrimitiveIndirect 引数 |

---

## ClearIndirectArgInstanceCountCS（BuildInstanceDrawCommands.usf:363）

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void ClearIndirectArgInstanceCountCS(
    uint3 GroupId          : SV_GroupID,
    int   GroupThreadIndex : SV_GroupIndex)
```

| 項目 | 内容 |
|-----|------|
| **目的** | DrawIndirect 引数の InstanceCount を 0 にリセット（毎フレーム実行）|
| **実行タイミング** | `InstanceCullBuildInstanceIdBufferCS` の前に必ず実行 |

---

## InstanceCullingLoadBalancer_Setup（InstanceCullingLoadBalancer.ush）

```hlsl
FInstanceWorkSetup InstanceCullingLoadBalancer_Setup(
    uint3 GroupId,
    int GroupThreadIndex,
    uint ExtraItems)
// GroupId + GroupThreadIndex からインスタンスバッチアイテムを逆引きする
// 戻り値: FInstanceWorkSetup { bValid, Item, LocalItemIndex }
```

---

## FMeshDrawCommand での使用

```cpp
// CPU 側の DrawIndirect 呼び出し（RendererScene.cpp）
RHICmdList.DrawIndexedPrimitiveIndirect(
    MeshBatch.VertexFactory->GetIndexBuffer(),
    DrawIndirectArgsBuffer,  // ← InstanceCullBuildInstanceIdBufferCS が生成
    DrawCommandOffset);
// GPU: VisibleInstanceCount 分だけインデックス付き描画
```

---

## 主要フラグ（INSTANCE_SCENE_DATA_FLAG_*）

| フラグ | 値 | 説明 |
|-------|------|------|
| `INSTANCE_SCENE_DATA_FLAG_HIDDEN` | `0x80000000` | 非表示 |
| `INSTANCE_SCENE_DATA_FLAG_CAST_SHADOW` | `0x00000001` | シャドウキャスト |
| `INSTANCE_SCENE_DATA_FLAG_RECEIVE_DECALS` | `0x00000002` | デカール受け取り |

---

> [!note]- GPU Driven Rendering の動作モデル
> GPU Driven Rendering 有効時（`r.GPUScene=1`）は：  
> 1. CPU は `DrawIndexedPrimitiveIndirect` コマンドのみを事前生成（メッシュごと1コマンド）  
> 2. GPU の `InstanceCullBuildInstanceIdBufferCS` がフラスタムカリングを実行  
> 3. 通過したインスタンスのみ `InstanceIdBuffer` に書き込み、`InstanceCount` をカウント  
> 4. `DrawIndexedPrimitiveIndirect` がそのカウント分だけ VS を起動  
> 従来の CPU カリングと比較して、数十万インスタンスでも高効率に処理できる。
