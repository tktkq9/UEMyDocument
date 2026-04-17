# REF: GPUScene Upload シェーダー

- グループ: a - Upload
- 詳細: [[detail_upload]]
- CPU リファレンス: [[ref_gpuscene_instance_culling]]
- ソース: `Engine/Shaders/Private/GPUScene/GPUSceneDataManagement.usf`  
          `Engine/Shaders/Private/GPUScene/GPUSceneWriter.ush`

---

## GPUSceneSetInstancePrimitiveIdCS（GPUSceneDataManagement.usf:11）

### エントリポイント

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void GPUSceneSetInstancePrimitiveIdCS(
    uint3 GroupId          : SV_GroupID,
    int   GroupThreadIndex : SV_GroupIndex)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Compute Shader |
| **スレッドグループ** | NUM_THREADS_PER_GROUP = 64（1D DispatchGroups）|
| **目的** | Instance の PrimitiveId と Flags を GPU バッファに書き込む |

#### ワークアイテム構造

```hlsl
// FInstanceWorkSetup（InstanceCullingLoadBalancer.ush）
struct FInstanceWorkSetup
{
    bool bValid;               // 有効なワークアイテムか
    FInstanceBatchItem Item;   // バッチアイテム情報
    uint LocalItemIndex;       // バッチ内ローカルインデックス
};

struct FInstanceBatchItem
{
    uint InstanceDataOffset;  // 開始 InstanceId
    uint Payload;             // PrimitiveId
    uint NumInstances;        // インスタンス数
};
```

#### CPU バインド

```cpp
// GPUScene.cpp - FGPUScene::Update()
FGPUSceneSetInstancePrimitiveIdCS::FParameters* PassParameters = ...;
PassParameters->InstanceCullingLoadBalancerParameters = LoadBalancer.GetShaderParameters();
PassParameters->GPUSceneWriteParameters = GPUSceneWriteParameters;

FComputeShaderUtils::AddPass(
    GraphBuilder,
    RDG_EVENT_NAME("GPUSceneSetInstancePrimitiveId"),
    ComputeShader, PassParameters,
    FIntVector(LoadBalancer.GetNumBatches(), 1, 1));
```

---

## GPUSceneWriter.ush 主要関数

```hlsl
// InstanceDataBuffer への書き込み
void WriteInstancePrimitiveIdAndFlags(
    uint InstanceId,
    uint PrimitiveId,
    uint InstanceFlags)
{
    uint PackedValue = (InstanceFlags << INSTANCE_SCENE_DATA_FLAG_SHIFT) | PrimitiveId;
    RWInstanceSceneData[InstanceId].PrimitiveIdAndFlags = PackedValue;
}

// INSTANCE_SCENE_DATA_FLAG_HIDDEN = 0x80000000  // 最上位ビット = 非表示フラグ
```

---

## FPrimitiveSceneShaderData GPU バッファレイアウト

```hlsl
// GetPrimitiveData() (SceneData.ush) でアクセス
FPrimitiveSceneData GetPrimitiveData(uint PrimitiveId)
{
    // GPUScene.PrimitiveBuffer[PrimitiveId * PRIMITIVE_SCENE_DATA_STRIDE + field]
    FPrimitiveSceneData Data;
    Data.LocalToWorld  = LoadRow(PrimitiveId, PRIMITIVE_SCENE_DATA_OFFSET_LOCAL_TO_WORLD);
    Data.ObjectBounds  = LoadRow(PrimitiveId, PRIMITIVE_SCENE_DATA_OFFSET_OBJECT_BOUNDS);
    Data.Flags         = asuint(LoadRow(PrimitiveId, PRIMITIVE_SCENE_DATA_OFFSET_FLAGS).x);
    ...
    return Data;
}
```

---

> [!note]- Dirty マーキングによる差分更新
> GPUScene は全プリミティブを毎フレーム転送せず、変更されたもの（Dirty）だけを転送する。  
> `FGPUScene::AddPrimitiveToUpdate()` で Dirty リストに追加し、  
> `FGPUScene::Update()` でそのリストの分だけ CS をディスパッチする。  
> 新規追加・変換変更・マテリアル変更などが Dirty のトリガーになる。
