# REF: Nanite Extensions シェーダー

- グループ: e - Extensions
- 詳細: [[detail_extensions]]
- CPU リファレンス: [[ref_nanite_ray_tracing]] | [[ref_nanite_stream_out]] | [[ref_nanite_feedback]]
- ソース: `Engine/Shaders/Private/Nanite/NaniteStreaming.usf`
          `Engine/Shaders/Private/Nanite/NaniteTranscode.usf`
          `Engine/Shaders/Private/Nanite/NaniteScatterUpdates.usf`
          `Engine/Shaders/Private/Nanite/NaniteRayTracing.usf`
          `Engine/Shaders/Private/Nanite/NaniteStreamOut.usf`
          `Engine/Shaders/Private/Nanite/NaniteEmitShadow.usf`

---

## NaniteStreaming.usf

### ClearStreamingRequestCount

```hlsl
[numthreads(1, 1, 1)]
void ClearStreamingRequestCount()
```

| 目的 | フレーム開始時にストリーミング要求カウンタ（`OutStreamingRequests[0]`）をリセット |
| CPU 関数 | `FStreamingManager::UpdateStreaming()` の先頭ステップ |

### Memcpy

```hlsl
[numthreads(64, 1, 1)]
void Memcpy(uint3 GroupID : SV_GroupID, uint GroupIndex : SV_GroupIndex)
```

| 項目 | 内容 |
|-----|------|
| **目的** | GPU 上でバイトバッファ間を 16 バイト単位で高速コピー（ページデータ転送）|
| **入力** | `SrcOffset`, `DstOffset`, `NumThreads`, `DstBuffer`（ByteAddressBuffer）|
| **CPU 関数** | `FStreamingManager::UpdateStreaming()` |

---

## NaniteTranscode.usf

### TranscodePageToGPU

```hlsl
[numthreads(GROUP_SIZE, 1, 1)]
void TranscodePageToGPU(uint3 GroupID : SV_GroupID, uint GroupIndex : SV_GroupIndex)
```

| 項目 | 内容 |
|-----|------|
| **目的** | ディスク圧縮フォーマットのクラスターページデータを GPU ランタイムフォーマット（`ClusterPageData`）に変換 |
| **パーミュテーション** | `NANITE_TRANSCODE_PASS_INDEPENDENT`（0）/ `NANITE_TRANSCODE_PASS_PARENT_DEPENDENT`（1）|
| **入力** | `StartClusterIndex`, `NumClusters`, ページバッファ（ByteAddressBuffer）|
| **出力** | `ClusterPageData`（GPU Scene の Nanite クラスターデータ）|
| **CPU 関数** | `FStreamingManager::UpdateStreaming()` |

#### 2パス処理

| パス | 内容 |
|------|------|
| Independent | 親クラスターへの参照が不要なデータ（頂点位置、クラスタ境界など）|
| ParentDependent | 親クラスターの頂点を参照して差分デコードするデータ |

---

## NaniteScatterUpdates.usf

### ScatterUpdates

```hlsl
[numthreads(64, 1, 1)]
void ScatterUpdates(uint3 GroupID : SV_GroupID, uint GroupIndex : SV_GroupIndex)
```

| 項目 | 内容 |
|-----|------|
| **目的** | `ScatterData`（DstOffset, Value ペアのリスト）を元に GPU Scene バッファへ散在書き込み |
| **入力** | `ScatterData`（ByteAddressBuffer）, `NumScatters`（書き込み件数）|
| **出力** | `DstBuffer`（RWByteAddressBuffer）— GPU Scene の各種テーブル |
| **CPU 関数** | `FNaniteScene::UpdateScatterBuffers()` |

---

## NaniteRayTracing.usf

### NaniteRayTracingStreamingTraversalCS

```hlsl
[numthreads(NANITE_PERSISTENT_CLUSTER_CULLING_GROUP_SIZE, 1, 1)]
void NaniteRayTracingStreamingTraversalCS(
    uint GroupID    : SV_GroupID,
    uint GroupIndex : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | HW Ray Tracing BLAS 構築のために必要なクラスターを Nanite BVH からトラバーサルして選定 |
| **パーミュテーション** | `CULLING_TYPE`: NODES / CLUSTERS |
| **コールバック** | `FNaniteTraversalRayTracingStreamingCallback` — 選定クラスターを出力バッファへ Append |
| **CPU 関数** | `FNaniteRayTracing::UpdateRayTracingScene()` |

### InitQueueCS（Ray Tracing 版）

```hlsl
[numthreads(64, 1, 1)]
void InitQueueCS(uint3 GroupId : SV_GroupID, int GroupThreadIndex : SV_GroupIndex)
```

| 目的 | Ray Tracing BVH トラバーサル前にキュー（`QueueState`, `Nodes`）を初期化。インスタンスごとにルートノードを設定 |
| CPU 関数 | `FNaniteRayTracing::UpdateRayTracingScene()` |

---

## NaniteStreamOut.usf

### NaniteStreamOutTraversalCS

```hlsl
[numthreads(NANITE_PERSISTENT_CLUSTER_CULLING_GROUP_SIZE, 1, 1)]
void NaniteStreamOutTraversalCS(
    uint GroupID    : SV_GroupID,
    uint GroupIndex : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Stream Out 対象クラスターを Nanite BVH からトラバーサルして選定 |
| **コールバック** | `FNaniteTraversalStreamOutCallback` — 選定クラスターを候補バッファへ Append |
| **CPU 関数** | `FStreamOutRequests::DispatchWork()` |

### InitQueue（Stream Out 版）

```hlsl
[numthreads(64, 1, 1)]
void InitQueue(uint GroupIndex : SV_GroupIndex, uint3 GroupId : SV_GroupID)
```

| 目的 | Stream Out 要求リスト（`StreamOutRequests`）を読み込んで頂点・インデックス範囲を確保（`AllocateRangesCommon`）|

### AllocateRangesCS

```hlsl
[numthreads(64, 1, 1)]
void AllocateRangesCS(uint GroupIndex : SV_GroupIndex, uint3 GroupId : SV_GroupID)
```

| 目的 | `ALLOCATE_VERTICES_AND_TRIANGLES_RANGES=1` 時: 各要求に頂点・インデックスバッファ領域を割り当て |

### NaniteStreamOutCS

```hlsl
[numthreads(STREAM_OUT_GROUP_SIZE, 1, 1)]
void NaniteStreamOutCS(
    uint GroupIndex       : SV_GroupIndex,
    uint3 GroupId         : SV_GroupID,
    uint3 DispatchThreadId : SV_DispatchThreadID
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 選定されたクラスターの頂点座標・法線・UV・インデックスを CPU 可読バッファに書き出す |
| **内部関数** | `WriteVertices()`, `WriteTriangles()`, `WriteSegment()`, `StreamOutClusterCommon()` |
| **出力** | `RWVertexBuffer`（頂点データ）, `RWIndexBuffer`（インデックスデータ）, `RWAuxiliaryDataBuffer` |
| **CPU 関数** | `FStreamOutRequests::DispatchWork()` |

---

## NaniteEmitShadow.usf

### EmitShadowMapPS

```hlsl
void EmitShadowMapPS(
    in float4 SvPosition : SV_Position,
    out float OutDepth   : SV_Depth
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Nanite VisBuffer から深度を読み取り Shadow Map RT に出力 |
| **パーミュテーション** | `DEPTH_OUTPUT_TYPE`: 0=標準, 1=正射投影, 2=透視投影（`DepthBias` 適用）|
| **入力** | `DepthBuffer`（Nanite 深度 uint）, `SourceOffset`, `ViewToClip22`, `DepthBias` |
| **CPU 関数** | `RenderShadowDepthMap()`（Nanite パス）|

### EmitCubemapShadowVS

```hlsl
void EmitCubemapShadowVS(
    uint VertexId               : SV_VertexID,
    out FEmitCubemapShadowVSOut Output
)
```

| 目的 | ポイントライト Cubemap Shadow 用 VS — Triangle 頂点を Cubemap フェース座標に変換 |

### EmitCubemapShadowGS

```hlsl
void EmitCubemapShadowGS(
    triangle FEmitCubemapShadowVSOut Input[3],
    inout TriangleStream<FEmitCubemapShadowGSOut> OutStream
)
```

| 目的 | 1三角形を `CubemapFaceIndex` に対応する Cubemap 面に GS で配置・出力 |

### EmitCubemapShadowPS

```hlsl
void EmitCubemapShadowPS(FEmitCubemapShadowGSOut In, out float OutDepth : SV_Depth)
```

| 目的 | Cubemap Shadow 用 PS — Nanite 深度バッファを参照して Shadow Map 深度を書き込む |

---

> [!note]- TranscodePageToGPU の2パス設計
> Nanite のクラスターデータは圧縮時に「親クラスターからの差分」を含む場合がある。  
> `PARENT_DEPENDENT` パスでは、先に `INDEPENDENT` パスが完了した後でないと親データが確定しないため、  
> 2パスを別々の CS ディスパッチで実行し、間に GPU バリアを挟む。  
> これにより1フレームあたり最大数百クラスターをオーバーラップなく安全にデコードできる。

> [!note]- NaniteStreamOut の用途と注意点
> Stream Out は GPU → CPU にメッシュデータを取り出す機能で、  
> プロシージャルジオメトリのコリジョン生成、PhysX/Chaos への LOD フィード、エディタのメッシュベイクなどに使用する。  
> 通常フレームレンダリングパスとは独立して `FStreamOutRequests` がバッチ処理で発行するため、  
> メインパスのパフォーマンスには直接影響しない（ただし GPU メモリ帯域を消費する）。

> [!note]- Emit Shadow と通常 Nanite パスの違い
> 通常の Nanite レンダリングは VisBuffer → Shade Binning → GBuffer の流れだが、  
> Shadow Map 生成では GBuffer への書き込み��不要で深度値のみが必要。  
> `EmitShadowMapPS` は VisBuffer の深度を直接 SV_Depth として出力することで、  
> マテリアル評価を完全にスキップして Shadow Map を高速生成する。
