# リファレンス：FGPUSceneResourceParameters / シェーダーバインド

- グループ: Reference
- 上位: [[09_gpuscene_overview]]
- 関連: [[ref_gpuscene_class]] | [[b_gpuscene_update]]
- ソース: `Engine/Source/Runtime/Renderer/Private/GPUScene.h`

---

## 概要

`FGPUSceneResourceParameters` は GPUScene バッファをシェーダーへバインドする  
`SHADER_PARAMETER_STRUCT` 定義。  
`SceneUniformBuffer` 経由で全シェーダーに自動的にバインドされる。

---

## FGPUSceneResourceParameters — シェーダーパラメータ

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FGPUSceneResourceParameters, )
    // プリミティブシェーダーデータ（行列・バウンド・マテリアル ID 等）
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, GPUScenePrimitiveSceneData)
    // インスタンスごとのローカル→ワールド変換
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, GPUSceneInstanceSceneData)
    // カスタムペイロード（プリミティブ種別ごとの拡張データ）
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, GPUSceneInstancePayloadData)
    // ライトマップ UV・スケール係数
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, GPUSceneLightmapData)
    // ライト全種のデータ（ByteAddressBuffer）
    SHADER_PARAMETER_RDG_BUFFER_SRV(ByteAddressBuffer, GPUSceneLightData)
    // インスタンスデータの SOA ストライド（タイル配置時）
    SHADER_PARAMETER(uint32, GPUSceneInstanceDataSOAStride)
    // 現在のフレーム番号（シェーダー内の最新チェック用）
    SHADER_PARAMETER(uint32, GPUSceneFrameNumber)
    // 割り当て済みインスタンスの最大 ID
    SHADER_PARAMETER(int32,  GPUSceneMaxAllocatedInstanceId)
    // 永続プリミティブの最大インデックス
    SHADER_PARAMETER(int32,  GPUSceneMaxPersistentPrimitiveIndex)
END_SHADER_PARAMETER_STRUCT()
```

---

## シェーダーパラメータ一覧

| パラメータ | シェーダー型 | 説明 |
|-----------|------------|------|
| `GPUScenePrimitiveSceneData` | `StructuredBuffer<float4>` | プリミティブデータ（行列・バウンド・フラグ等）|
| `GPUSceneInstanceSceneData` | `StructuredBuffer<float4>` | インスタンスごとのローカル→ワールド変換 |
| `GPUSceneInstancePayloadData` | `StructuredBuffer<float4>` | カスタムペイロード（プリミティブ種別依存）|
| `GPUSceneLightmapData` | `StructuredBuffer<float4>` | ライトマップ UV スケール・バイアス |
| `GPUSceneLightData` | `ByteAddressBuffer` | ライトシーンデータ（FLightSceneData）|
| `GPUSceneInstanceDataSOAStride` | `uint32` | SOA タイル配置のストライド（通常 = 0）|
| `GPUSceneFrameNumber` | `uint32` | Update 実行時のフレーム番号 |
| `GPUSceneMaxAllocatedInstanceId` | `int32` | 最大インスタンス ID（上限チェック用）|
| `GPUSceneMaxPersistentPrimitiveIndex` | `int32` | 永続プリミティブ上限（動的判定用）|

---

## シェーダーからのアクセス方法

```hlsl
// シェーダー内（Scene.ush が SceneUniformBuffer 経由で提供）
float4 PrimData0 = GPUScenePrimitiveSceneData[PrimitiveId * PRIMITIVE_SCENE_DATA_STRIDE + 0];
float4x4 LocalToWorld = GetPrimitiveData(PrimitiveId).LocalToWorld;

float4 InstanceData = GPUSceneInstanceSceneData[InstanceId * INSTANCE_SCENE_DATA_STRIDE + 0];
```

アクセスは `SceneData.ush` / `GPUSceneWriter.ush` のヘルパーマクロ経由が標準。

---

## SceneUniformBuffer へのバインド

```
FGPUScene::FillSceneUniformBuffer(GraphBuilder, SceneUB)
  └─ SceneUB.Set<FSceneUniformParameters::GPUScene>(GetShaderParameters(GraphBuilder))
       └─ 全パスで SetShaderParametersLambda() → 自動バインド
```

`GetShaderParameters(GraphBuilder)` が `FGPUSceneResourceParameters` を構築して返す。  
`UpdateInternal()` 完了後に `FillSceneUniformBuffer()` が呼ばれ、  
以降の全シェーダー（Nanite / Lumen / VSM / RayTracing / BasePass）が最新バッファを参照する。

---

> [!note]- GPUSceneMaxPersistentPrimitiveIndex による動的プリミティブ判定
> シェーダー内で `PrimitiveId >= GPUSceneMaxPersistentPrimitiveIndex` の場合は動的プリミティブ。  
> 動的プリミティブは毎フレーム `UploadDynamicPrimitiveShaderDataForView()` でアップロードされる。  
> 永続プリミティブと動的プリミティブを同じバッファに配置することで、  
> シェーダーは ID だけを見ればよく、分岐なしで両方を参照できる。

> [!note]- GPUSceneInstanceDataSOAStride と タイル配置
> 通常（SOA モードオフ）は `GPUSceneInstanceDataSOAStride = 0`。  
> `r.GPUScene.InstanceDataTileSizeLog2 >= 0` でタイルモードが有効になると  
> インスタンスデータが SoA (Structure of Arrays) タイル配置に切り替わり、  
> `SOAStride` にタイルサイズが入る。キャッシュ効率向上が目的だが実験的機能。

> [!note]- FGPUSceneResourceParameters vs FSceneUniformParameters
> `FGPUSceneResourceParameters` は GPUScene バッファのみを宣言するサブ構造体。  
> `FSceneUniformParameters`（SceneUniformBuffer 全体）のメンバとして `GPUScene` フィールドに入っている。  
> シェーダー内では `View.GPUScene.GPUScenePrimitiveSceneData` のようにアクセスするのではなく、  
> `GPUScene.ush` が提供する `GetPrimitiveData()` / `GetInstanceSceneData()` ヘルパーを使うのが標準。
