# Nanite Extensions シェーダー詳細

- グループ: e - Extensions
- GPU 概要: [[01_nanite_gpu_overview]]
- CPU 詳細: [[d_nanite_ray_tracing]] | [[f_nanite_debug_editor]]
- リファレンス: [[ref_extensions]]

---

## 概要

メイン Nanite パイプライン（カリング → ラスタライズ → シェーディング → 深度出力）とは独立して実行される  
拡張パスのシェーダー群。

| パス | シェーダー | 概要 |
|-----|----------|------|
| Streaming | `NaniteStreaming.usf` | GPU フィードバック収集 + ページデータのコピー |
| Transcode | `NaniteTranscode.usf` | ストリーミングページを GPU アクセス可能なフォーマットに変換 |
| Scatter Updates | `NaniteScatterUpdates.usf` | GPU Scene への散在書き込み |
| Ray Tracing | `NaniteRayTracing.usf` | BLAS 構築用 BVH トラバーサル |
| Stream Out | `NaniteStreamOut.usf` | Nanite クラスタ → CPU 可読頂点/インデックスバッファへ展開 |
| Emit Shadow | `NaniteEmitShadow.usf` | Nanite の Shadow Map 生成 |

---

## Streaming / Transcode

### GPU フィードバック → ストリーミング要求

Nanite はメインパスで参照したページを `OutStreamingRequests` に記録する。  
CPU 側の `FStreamingManager` がこれを読み取り、不足ページをディスクからロードする。

```
[GPU] クラスターカリング / ラスタライザ
  → OutStreamingRequests（参照したページID を Append）

[CPU] FStreamingManager::UpdateStreaming()
  → GPU Feedback を読み取り
  → ページデータをディスク / IOから読み込み
  → TranscodePageToGPU CS で GPU メモリに展開
  → ScatterUpdates CS で GPU Scene に書き込み
```

### ClearStreamingRequestCount

```hlsl
[numthreads(1, 1, 1)]
void ClearStreamingRequestCount()
{
    OutStreamingRequests[0].RuntimeResourceID_Magic = 0;  // カウンタリセット
}
```

フレーム開始時に前フレームの要求カウントをクリア。

### Memcpy（NaniteStreaming.usf）

```hlsl
[numthreads(64, 1, 1)]
void Memcpy(uint3 GroupID, uint GroupIndex)
```

GPU 上でのバイトバッファ間コピー。ページデータの GPU 転送に使用。

### TranscodePageToGPU（NaniteTranscode.usf）

```hlsl
[numthreads(GROUP_SIZE, 1, 1)]
void TranscodePageToGPU(uint3 GroupID : SV_GroupID, uint GroupIndex : SV_GroupIndex)
```

ディスクフォーマットのページデータ（圧縮された頂点・クラスター情報）を  
GPU がリアルタイムアクセスできる内部フォーマット（`ClusterPageData`）に変換する。

2パスで実行：
- `NANITE_TRANSCODE_PASS_INDEPENDENT`: 親ページに依存しないデータを先にトランスコード
- `NANITE_TRANSCODE_PASS_PARENT_DEPENDENT`: 親ページ参照が必要なデータを後処理

---

## Scatter Updates

```hlsl
[numthreads(64, 1, 1)]
void ScatterUpdates(uint3 GroupID : SV_GroupID, uint GroupIndex : SV_GroupIndex)
```

`ScatterData` バッファに格納された `(DstOffset, Value)` のペアを GPU Scene バッファに散在書き込みする。  
Nanite インスタンスの BVH データ更新（移動・変形後の AABB 更新等）に使用。

---

## Ray Tracing

Nanite は HW Ray Tracing 用の BLAS（Bottom Level Acceleration Structure）を持つために  
独自の BVH トラバーサル CS でクラスターを選定し、メッシュデータとして展開する。

### NaniteRayTracingStreamingTraversalCS

```hlsl
[numthreads(NANITE_PERSISTENT_CLUSTER_CULLING_GROUP_SIZE, 1, 1)]
void NaniteRayTracingStreamingTraversalCS(uint GroupID, uint GroupIndex)
```

Nanite の BVH をトラバーサルし、Ray Tracing BLAS 構築に必要なクラスターを選定する。  
`FNaniteTraversalRayTracingStreamingCallback` でクラスターごとの処理を行う。

```
NaniteRayTracingStreamingTraversalCS
  → FNaniteTraversalRayTracingStreamingCallback::ProcessCluster()
  → BLAS 構築対象クラスターリストへ Append
  → D3D12/Vulkan の BLAS Build に渡す（CPU 側）
```

### InitQueueCS

```hlsl
[numthreads(64, 1, 1)]
void InitQueueCS(uint3 GroupId, int GroupThreadIndex)
```

Ray Tracing BVH トラバーサル開始前にキュー状態を初期化。インスタンスごとに BVH ルートノードを設定する。

---

## Stream Out

Nanite クラスターを CPU が読める通常のメッシュ（頂点 + インデックスバッファ）として展開する機能。  
プロシージャルメッシュ生成・フィジクス等との連携に使用。

### NaniteStreamOutTraversalCS

```hlsl
[numthreads(NANITE_PERSISTENT_CLUSTER_CULLING_GROUP_SIZE, 1, 1)]
void NaniteStreamOutTraversalCS(uint GroupID, uint GroupIndex)
```

`FNaniteTraversalStreamOutCallback` で BVH をトラバーサルし、展開対象クラスターを選定。

### InitQueue（StreamOut）

```hlsl
[numthreads(64, 1, 1)]
void InitQueue(uint GroupIndex, uint3 GroupId)
```

Stream Out 要求リストを読み込み、頂点・インデックスバッファの確保範囲を計算 (`AllocateRangesCommon`)。

### AllocateRangesCS

```hlsl
[numthreads(64, 1, 1)]
void AllocateRangesCS(uint GroupIndex, uint3 GroupId)
```

各 Stream Out 要求（`FNaniteStreamOutRequest`）に対してバッファ範囲を割り当て。

### NaniteStreamOutCS

```hlsl
[numthreads(STREAM_OUT_GROUP_SIZE, 1, 1)]
void NaniteStreamOutCS(uint GroupIndex, uint3 GroupId, uint3 DispatchThreadId)
```

選定されたクラスターの頂点・三角形データを `WriteVertices()` / `WriteTriangles()` で  
CPU 可読バッファに書き込む。マテリアルセグメント別に出力位置を分割する。

---

## Emit Shadow

Nanite メッシュのシャドウマップを通常の PS パイプラインで出力するシェーダー群。  
`NaniteExportGBuffer.usf#EmitShadowMapPS` と連携。

### EmitShadowMapPS

```hlsl
void EmitShadowMapPS(in float4 SvPosition : SV_Position, out float OutDepth : SV_Depth)
```

Nanite VisBuffer から読んだ深度値を Shadow Map RT に出力。  
`DEPTH_OUTPUT_TYPE` で標準深度 / 正射投影影 / 透視投影影を切り替える。

### EmitCubemapShadowVS / GS / PS

```hlsl
void EmitCubemapShadowVS(uint VertexId : SV_VertexID, out FEmitCubemapShadowVSOut Output)
void EmitCubemapShadowGS(triangle ..., inout TriangleStream<...> OutStream)
void EmitCubemapShadowPS(FEmitCubemapShadowGSOut In, out float OutDepth : SV_Depth)
```

ポイントライトの Cubemap Shadow Map 生成。GS で6面に三角形を複製する。

---

## CPU 呼び出しの流れ

```
FStreamingManager::UpdateStreaming()       // NaniteStreamingManager.cpp
  ├─ ClearStreamingRequestCount CS
  ├─ [メインパス実行中に OutStreamingRequests に記録]
  ├─ Memcpy CS（ページデータ転送）
  └─ TranscodePageToGPU CS × 2（独立パス→親依存パス）

ScatterUpdates CS                          // NaniteScatterUpdates.usf
  └─ GPU Scene BVH データの散在書き込み

FNaniteRayTracing::UpdateRayTracingScene() // NaniteRayTracing.cpp
  ├─ InitQueueCS（Ray Tracing 版）
  └─ NaniteRayTracingStreamingTraversalCS

FStreamOutRequests::DispatchWork()         // NaniteStreamOut.cpp
  ├─ InitQueue CS
  ├─ AllocateRangesCS
  ├─ NaniteStreamOutTraversalCS
  └─ NaniteStreamOutCS

RenderShadowDepthMap()（Nanite パス）
  ├─ EmitShadowMapPS
  └─ EmitCubemapShadowVS/GS/PS
```
