# GPU a: Core — TLAS/BLAS 構築・組み込みシェーダー

- シェーダー: `RayTracing/RayTracingBuiltInShaders.usf`, `RayTracingInstanceBufferUtil.usf`, `RayTracingDynamicMesh.usf`
- CPU 対応: [[a_rt_scene]]
- 上位: [[01_raytracing_gpu_overview]]

---

## 概要

TLAS（Top Level Acceleration Structure）の構築に使う GPU バッファ書き込み CS と、  
テスト・フォールバック用の組み込みシェーダー群。  
RHI の BuildAccelerationStructure 呼び出しは CPU 側が発行し、  
GPU は CS で**インスタンスバッファ**を準備するだけ。

---

## BuildRayTracingInstanceBufferCS（RayTracingInstanceBufferUtil.usf: line 226）

CPU が収集したインスタンスリストを GPU バッファ（D3D12_RAYTRACING_INSTANCE_DESC 相当）に書き込む。

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void BuildRayTracingInstanceBufferCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // 1. DispatchThreadId.x = インスタンスインデックス
    // 2. InstanceInputBuffer から FRayTracingInstanceDescInput を読み込み
    //    → Transform（3×4 行列）, MaskFlags, InstanceID, GeometryIndex
    // 3. D3D12_RAYTRACING_INSTANCE_DESC フォーマットに変換
    // 4. InstanceOutputBuffer[InstanceIndex] に書き込み
    //    → RHI の TLAS Build に使用される
}
```

**パラメータ:**

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `InstanceInputBuffer` | `StructuredBuffer<FRayTracingInstanceDescInput>` | CPU 側が準備したインスタンス情報 |
| `InstanceOutputBuffer` | `RWStructuredBuffer<uint4>` | D3D12 インスタンス記述子バッファ |
| `NumInstances` | `uint` | インスタンス総数 |

---

## RayTracingDynamicGeometryConverterCS（RayTracingDynamicMesh.usf: line 45）

スケルタルメッシュ等の動的ジオメトリの頂点データを RT 用バッファ（BLAS 入力）に変換する。

```hlsl
[numthreads(64, 1, 1)]
void RayTracingDynamicGeometryConverterCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // 頂点バッファから Position を読み込み
    // → BLAS ビルド用のフラット頂点バッファに書き出し
}
```

---

## 組み込みシェーダー（RayTracingBuiltInShaders.usf）

テスト・デバッグ・デフォルトフォールバック用の組み込みシェーダー群。

### OcclusionMainRGS（line 12）

TLAS の接続テスト用 RGS。各ピクセルから1本レイを発射し Hit/Miss を確認する。

```hlsl
RAY_TRACING_ENTRY_RAYGEN(OcclusionMainRGS)
{
    // TraceOcclusionRay() で1本レイを発射
    // → OcclusionOutput に 0.0(Hit) / 1.0(Miss) を書き込み
}
```

### IntersectionMainRGS（line 47）

交差テスト用 RGS。ヒット座標・法線をバッファに出力する。

### IntersectionMainCHS（line 76）

交差テスト用 Closest Hit Shader。`FIntersectionPayload` にヒット情報を書き込む。

### DefaultMainCHS（line 86）

デフォルト Closest Hit Shader。マテリアル評価をスキップし `IsMiss()` フラグのみ立てる。

### DefaultOpaqueAHS（line 97）

デフォルト Any Hit Shader。不透明オブジェクト用（処理なし・即 Accept）。

### DefaultPayloadMS（line 103）

`FDefaultPayload` 用 Miss Shader。`IsMiss()` フラグを立てるだけ。

### PackedMaterialClosestHitPayloadMS（line 108）

`FPackedMaterialClosestHitPayload` 用 Miss Shader。マテリアル評価系の Miss ハンドラ。

---

## RayTracingLightingMS（RayTracingLightingMS.usf: line 55）

マテリアルシェーダーと組み合わせて使う **Miss Shader**。  
レイがどこにもヒットしない場合（スカイやエミッシブなし方向）に環境ライティングを評価する。

```hlsl
RAY_TRACING_ENTRY_MISS(RayTracingLightingMS, FPackedMaterialClosestHitPayload, PackedPayload)
{
    // Sky Light / Reflection Capture の評価
    // → PackedPayload に環境光の色を書き込み
}
```

---

## TLAS 構築フロー（CPU + GPU の協調）

```
[CPU] FRayTracingScene::Update()
  → FRayTracingInstanceDescInput バッファを RDG に登録

[GPU] BuildRayTracingInstanceBufferCS
  → D3D12_RAYTRACING_INSTANCE_DESC 形式に変換
  → RHI の TLAS バッファに書き込み

[CPU] FRayTracingScene::Build()
  → RDG パスで BuildAccelerationStructure (RHI) を呼び出し
  → TLAS が GPU メモリ上に確定

[後続の RGS]
  → TLAS を RaytracingAccelerationStructure として参照
  → TraceRay() / TraceRayInline() でトラバーサル
```
