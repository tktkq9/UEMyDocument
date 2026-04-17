# GPUScene Upload シェーダー詳細

- グループ: a - Upload
- GPU 概要: [[01_gpuscene_gpu_overview]]
- CPU 詳細: [[a_gpuscene_upload]]
- リファレンス: [[ref_upload]]

---

## 概要

CPU 上で更新された **PrimitiveSceneData** と **InstanceSceneData** を GPU 常駐バッファに書き込む。  
毎フレーム変更されたプリミティブのみ差分更新（Dirty マーキング方式）で転送コストを最小化する。

---

## レンダリングパスの構成

```
GPUScene Upload
  │
  ├─ [Primitive データ転送]
  │   CPU バッファ（FPrimitiveSceneShaderData）→ GPU バッファ（PrimitiveBuffer）
  │   変更されたプリミティブのみ Upload（FGPUScene::UpdatedPrimitiveIds）
  │
  └─ [Instance データ転送]
      CS: GPUSceneSetInstancePrimitiveIdCS()
      InstanceId バッファの PrimitiveId フィールドを書き込み
      INSTANCE_SCENE_DATA_FLAG_HIDDEN フラグで不可視を設定
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `InstanceCullingLoadBalancer` | 更新すべき InstanceId のリスト |
| `WorkSetup.Item.Payload` | 書き込む PrimitiveId |

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| `GPUScene.InstanceDataBuffer` | StructuredBuffer | InstanceSceneData（PrimitiveId + Flags）|
| `GPUScene.PrimitiveBuffer` | StructuredBuffer | FPrimitiveSceneShaderData 全フィールド |

---

## シェーダーコアロジック（GPUSceneDataManagement.usf）

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void GPUSceneSetInstancePrimitiveIdCS(
    uint3 GroupId       : SV_GroupID,
    int   GroupThreadIndex : SV_GroupIndex)
{
    // InstanceCullingLoadBalancer からワークアイテムを取得
    FInstanceWorkSetup WorkSetup = InstanceCullingLoadBalancer_Setup(GroupId, GroupThreadIndex, 0U);

    if (!WorkSetup.bValid) return;

    // InstanceId を計算して PrimitiveId + Flags を書き込む
    const uint InstanceId    = WorkSetup.Item.InstanceDataOffset + uint(WorkSetup.LocalItemIndex);
    const uint PrimitiveId   = WorkSetup.Item.Payload;
    const uint InstanceFlags = INSTANCE_SCENE_DATA_FLAG_HIDDEN;  // 初期は非表示

    WriteInstancePrimitiveIdAndFlags(InstanceId, PrimitiveId, InstanceFlags);
    // → GPUSceneWriter.ush: InstanceDataBuffer[InstanceId].PrimitiveId = PrimitiveId
}
```

---

## FPrimitiveSceneShaderData（CPU→GPU 転送データ）

```cpp
// CPU 側: FPrimitiveSceneShaderData（ScenePrivate.h）
// GPU 側: GetPrimitiveData(uint PrimitiveId) で参照
struct FPrimitiveSceneShaderData
{
    // Transform matrices
    float4 LocalToWorld[3];      // 3×4 行列
    float4 WorldToLocal[3];
    float4 PrevLocalToWorld[3];  // 前フレーム変換

    // Bounds
    float4 ObjectWorldPositionAndRadius;  // xyz=中心, w=半径
    float4 ObjectBounds;

    // Material / Lighting
    float4 LightmapUVScaleBias;
    float4 PrecomputedShadingParameters;

    // Flags
    uint PrimitiveId;
    uint Flags;          // 可視・キャスト影・受影等
    // ... 多数のフィールド
};
```

---

## CPU 呼び出しの流れ

```
FGPUScene::Update()                          // GPUScene.cpp
  │
  ├─ UpdatedPrimitiveIds（Dirty リスト）を走査
  ├─ FPrimitiveSceneShaderData を CPU 側で構築
  │   → VertexBuffer（Upload Buffer）に書き込み
  │
  └─ AddPass(RDG_EVENT_NAME("GPUSceneUpload"))
      CS: GPUSceneSetInstancePrimitiveIdCS()
      各インスタンスの PrimitiveId + Flags を GPU に反映

      RDGバッファ:
        PrimitiveBuffer     → DrawMeshCommands 側で GetPrimitiveData() で参照
        InstanceDataBuffer  → Instance Culling で可視判定に使用
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `GPUScene/GPUSceneDataManagement.usf` | CS | InstanceSceneData 書き込み |
| `GPUScene/GPUSceneWriter.ush` | ヘッダ | `WriteInstancePrimitiveIdAndFlags()` 等書き込みユーティリティ |
| `InstanceCulling/InstanceCullingLoadBalancer.ush` | ヘッダ | ワークアイテム分散ユーティリティ |
