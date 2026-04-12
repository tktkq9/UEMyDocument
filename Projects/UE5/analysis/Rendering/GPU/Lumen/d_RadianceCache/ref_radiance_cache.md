# REF: Lumen Radiance Cache シェーダー

- グループ: d - Radiance Cache
- 詳細: [[detail_radiance_cache]]
- CPU リファレンス: [[ref_lumen_radiance_cache]]
- ソース: `Engine/Shaders/Private/Lumen/LumenRadianceCache.usf`  
          `Engine/Shaders/Private/Lumen/LumenRadianceCacheUpdate.usf`  
          `Engine/Shaders/Private/Lumen/LumenRadianceCacheHardwareRayTracing.usf`

---

## LumenRadianceCacheUpdate.usf（プローブ管理）

### ClearRadianceCacheUpdateResourcesCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void ClearRadianceCacheUpdateResourcesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | 更新バッファをフレーム冒頭でゼロクリア |

### AllocateUsedProbesCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, THREADGROUP_SIZE)]
void AllocateUsedProbesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | Screen Probe に参照されたプローブを「使用中」としてマーク・アロケーション |
| 出力 | `RWProbeAllocator`, `RWProbeLastUsedFrame` |

### SelectMaxPriorityBucketCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void SelectMaxPriorityBucketCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | 更新バジェット内で最大優先度バケットを選択（DirectLighting と同様の予算制御）|

### AllocateProbeTracesCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, THREADGROUP_SIZE)]
void AllocateProbeTracesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | 更新対象プローブにトレーススロットを割り当て |
| 出力 | `RWProbeTraceAllocator`, `RWProbeTraceData` |

### UpdateCacheForUsedProbesCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, THREADGROUP_SIZE)]
void UpdateCacheForUsedProbesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | 使用中プローブの位置・優先度をキャッシュに反映 |

---

## LumenRadianceCache.usf（トレース・フィルタ・合成）

### ClearProbeIndirectionCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, THREADGROUP_SIZE)]
void ClearProbeIndirectionCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | Probe の空間インデックステーブル（IndirectionTexture 3D）をクリア |

### ComputeProbeWorldOffsetsCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void ComputeProbeWorldOffsetsCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | プローブのジッタリング・ワールドオフセットを計算（時間的なばらつきによる品質改善）|

### ScatterScreenProbeBRDFToRadianceProbesCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ScatterScreenProbeBRDFToRadianceProbesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | Screen Probe が収集した BRDF 情報を Radiance Cache Probe に散布（重要度サンプリング用 PDF 生成）|
| 出力 | `RWDebugBRDFProbabilityDensityFunction`（デバッグ用 PDF テクスチャ）|

### GenerateProbeTraceTilesCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void GenerateProbeTraceTilesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | プローブごとのレイ方向タイル（2D）を生成 → 後のソート・トレースに使用 |

### SortProbeTraceTilesCS

```hlsl
[numthreads(SORT_TILES_THREADGROUP_SIZE, 1, 1)]
void SortProbeTraceTilesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | タイルを Material ID でソート（GPU ソート）→ 同じマテリアルのレイを連続実行してキャッシュ効率向上 |

### TraceFromProbesCS

```hlsl
[numthreads(RADIANCE_CACHE_TRACE_TILE_SIZE_2D, RADIANCE_CACHE_TRACE_TILE_SIZE_2D, 1)]
void TraceFromProbesCS(uint3 GroupId, uint3 GroupThreadId)
```
| 項目 | 内容 |
|-----|------|
| **目的** | ソート済みタイルのレイをトレース（SDF / HW RT）してプローブ輝度を記録 |
| **入力** | `FinalLightingAtlas`, `GlobalSDF`, `MeshSDFData` |
| **出力** | `RWRadianceProbeAtlasTexture`（float3 輝度）, `RWDepthProbeAtlasTexture`（float 深度）|
| **CPU 関数** | `RenderRadianceCacheTracing()` |

#### パーミュテーション

| クラス | 説明 |
|-------|------|
| `TRACE_GLOBAL_SDF` | Global SDF トレース使用 |
| `INDIRECT_DISPATCH` | Indirect Dispatch 形式 |

### FilterProbeRadianceWithGatherCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void FilterProbeRadianceWithGatherCS(uint3 GroupId, uint3 GroupThreadId)
```
| 目的 | Probe 輝度アトラスの空間フィルタリング（隣接 Probe 参照による品質向上）|
| 出力 | `RWRadianceProbeAtlasTexture`（フィルタ済み）|

### CalculateProbeIrradianceCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void CalculateProbeIrradianceCS(uint3 GroupId, uint3 GroupThreadId)
```
| 項目 | 内容 |
|-----|------|
| **目的** | 全方向輝度アトラスをコサイン加重で積分して Irradiance 生成 |
| **出力** | `RWFinalIrradianceAtlas`（Screen Probe の遠距離補完に使用）|

### PrepareProbeOcclusionCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void PrepareProbeOcclusionCS(uint3 GroupId, uint3 GroupThreadId)
```
| 目的 | プローブの遮蔽（Visibility Occlusion）情報を事前計算 |
| 出力 | `RWRadianceCacheProbeOcclusionAtlas`（UNORM float2）|

### FixupBordersAndGenerateMipsCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void FixupBordersAndGenerateMipsCS(uint3 GroupId, uint3 GroupThreadId)
```
| 目的 | アトラス境界のダイレーション修正 + ミップチェーン生成（Mip0/1/2）|
| 出力 | `RWFinalRadianceAtlasMip0/1/2`（補間・遠距離サンプリング用）|

---

## LumenRadianceCacheHardwareRayTracing.usf

```hlsl
RAY_TRACING_ENTRY_RAYGEN(LumenRadianceCacheHardwareRayTracingRGS)
```
| 目的 | HW RT でプローブレイをトレース（`r.Lumen.RadianceCache.HardwareRayTracing=1` 時）|
| CPU 関数 | `DispatchRadianceCacheHardwareRayTracing()` |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Lumen.RadianceCache.NumProbeTracesBudget` | 200 | 1フレームに更新するプローブ数上限 |
| `r.Lumen.RadianceCache.ProbeResolution` | 32 | プローブアトラスの解像度（32×32）|
| `r.Lumen.RadianceCache.HardwareRayTracing` | 0 | HW RT 使用 |
| `r.Lumen.RadianceCache.GridResolution` | 48 | 3D グリッドのセル数 |

---

> [!note]- Radiance Cache と Screen Probe の役割分担
> Screen Probe Gather は「ピクセルに近い数 m 〜数十 m のレイ」を毎フレーム打って最新の GI を得る。  
> Radiance Cache は「数十 m 〜数百 m の遠距離レイ」の代わりとして機能し、プローブを数フレームにわたりキャッシュすることでトレースコストを削減する。  
> Screen Probe トレース中に遠距離で SDF がミスした場合、`SampleRadianceCacheInterpolated()` でキャッシュから補完する。

> [!note]- PDF ソートによるトレースコスト削減
> `SortProbeTraceTilesCS()` は BRDF 重要度サンプリングで生成した PDF を使い、「同じマテリアルのレイ」を隣接スレッドにまとめる。  
> GPU ではテクスチャキャッシュのヒット率がスレッド間の参照局所性に依存するため、同一マテリアルのレイをグループ化することで FinalLightingAtlas へのアクセスが最適化される。
