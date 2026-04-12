# REF: Lumen SDF トレーシング シェーダー

- グループ: c - Tracing
- 詳細: [[detail_sdf_tracing]]
- CPU リファレンス: [[ref_lumen_tracing]]
- ソース: `Engine/Shaders/Private/Lumen/LumenMeshSDFCulling.usf`  
          `Engine/Shaders/Private/Lumen/LumenScene.usf`  
          `Engine/Shaders/Private/Lumen/LumenSoftwareRayTracing.ush`（共通ヘッダ）  
          `Engine/Shaders/Private/Lumen/LumenTracingCommon.ush`（共通ヘッダ）

---

## LumenMeshSDFCulling.usf

### CombineObjectIndexBuffersCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void CombineObjectIndexBuffersCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | 複数ソースのオブジェクトインデックスバッファを統合 |
|-----|------|

### CullHeightfieldObjectsForViewCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void CullHeightfieldObjectsForViewCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | Height Field SDF のビューカリング |
|-----|------|
| **出力** | `RWVisibleHeightfieldObjectsBuffer` |

### CullMeshSDFObjectsForViewCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void CullMeshSDFObjectsForViewCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | Mesh SDF オブジェクトのビュー + 距離カリング |
|-----|------|
| **入力** | `MeshSDFObjectBounds`, `ViewFrustumPlanes`, `MaxMeshSDFTraceDistance` |
| **出力** | `RWVisibleMeshSDFsBuffer`（可視インデックス）, `RWNumVisibleMeshSDFs` |
| **CPU 関数** | `CullMeshSDFObjectsForView()` / `CullMeshSDFsToProbes()` |

### MeshSDFObjectCullVS / MeshSDFObjectCullPS

```hlsl
void MeshSDFObjectCullVS(uint ObjectIndex : SV_InstanceID, uint VertexIndex : SV_VertexID, out float4 OutPosition : SV_POSITION)
void MeshSDFObjectCullPS(float4 SvPosition : SV_POSITION, uint ObjectIndex : SV_InstanceID)
```
| 目的 | Mesh SDF バウンディングボックスをラスタライズして視錐台カリング（ラスタライズベース）|
|-----|------|

### CompactCulledObjectsCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void CompactCulledObjectsCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | カリング後のスパース配列を密に Compact |
|-----|------|
| **出力** | `RWCompactedVisibleMeshSDFs`（連続配列）|

### ComputeCulledObjectsStartOffsetCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void ComputeCulledObjectsStartOffsetCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | Probe / Tile ごとの SDF 参照開始オフセットを計算（prefix sum）|
|-----|------|

---

## LumenScene.usf

### LumenSceneUpdateCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void LumenSceneUpdateCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 項目 | 内容 |
|-----|------|
| **目的** | CPU からアップロードされた Card / CardPage データを GPU バッファに反映 |
| **入力** | `UploadedCardData`, `UploadedCardPageData` |
| **出力** | `RWLumenCardData[]`, `RWLumenCardPageData[]`（GPU Scene バッファ）|
| **CPU 関数** | `UpdateLumenScene()` in `LumenSceneRendering.cpp` |

---

## LumenSoftwareRayTracing.ush（共通ヘッダ — 直接 include して使用）

### 主要関数

| 関数 | 戻り値 | 説明 |
|-----|--------|------|
| `TraceMeshSDF(RayOrigin, RayDir, MaxDist, SdfIndex)` | `FLumenMinHitResult` | 単一 Mesh SDF に対してレイをステップマーチ |
| `TraceGlobalSDF(RayOrigin, RayDir, MaxDist)` | `FLumenMinHitResult` | Global SDF（Voxel）でレイトレース |
| `GetDistanceToMeshSDF(WorldPos, SdfIndex)` | `float` | ワールド座標から SDF 距離を読み取り |

---

## LumenTracingCommon.ush（共通ヘッダ — 直接 include して使用）

### 主要構造体・関数

```hlsl
// ヒット結果
struct FLumenMinHitResult
{
    bool   bHit;
    float  HitDistance;
    float3 HitNormal;
    uint   CardIndex;     // Surface Cache の Card インデックス
    float2 CardUV;        // Card 上の UV（FinalLightingAtlas サンプリング用）
};

// Surface Cache からヒット輝度を取得
float3 SampleLumenCardData(FLumenMinHitResult HitResult, Texture2D FinalLightingAtlas, SamplerState Sampler);

// Global SDF から法線を再構築
float3 ComputeGlobalSDFNormal(float3 WorldPos, float VoxelSize);
```

---

## LumenHardwareRayTracingCommon.ush（HW RT 共通 — include して使用）

### 主要型

```hlsl
struct FLumenMinimalRayPayload   // 最小ペイロード（影判定用）
{
    float HitT;     // ヒット距離（-1 = Miss）
};

struct FLumenRayPayload          // フルペイロード（GI/反射用）
{
    float  HitT;
    uint   PackedColor;      // ヒット輝度（R11G11B10F パック）
    float3 WorldNormal;
    uint   MaterialShaderIndex;
};
```

---

> [!note]- Mesh SDF と Global SDF の使い分け
> Mesh SDF は近距離（概ね 100m 以内）で使われ、メッシュ形状に忠実なトレースが可能。ただしオブジェクト数が多いとカリングコストが増える。  
> Global SDF は低解像度 Voxel（例: 512³）で遠距離をカバーし、カリング不要でトレース 1回あたりのコストが安定している。  
> 実際のトレースは「Mesh SDF でできるだけ進み、カバー外になったら Global SDF に切り替える」2段階方式。

> [!note]- SDF カリングの Compact 化の意味
> `CullMeshSDFObjectsForViewCS()` の直後は「0か1かのフラグ配列」になっており、アクセスが非連続。  
> `CompactCulledObjectsCS()` でフラグ=1 のエントリだけを密な配列に詰め直すことで、後続のトレース CS が `[objectIndex]` を連番で走査でき、メモリアクセスの局所性が上がる。
