# REF: Lumen Reflection Tracing シェーダー

- グループ: f - Reflections
- 詳細: [[detail_reflections]]
- CPU リファレンス: [[ref_lumen_reflections]] | [[ref_lumen_reflection_tracing]]
- ソース: `Engine/Shaders/Private/Lumen/LumenReflections.usf`  
          `Engine/Shaders/Private/Lumen/LumenReflectionTracing.usf`  
          `Engine/Shaders/Private/Lumen/LumenReflectionResolve.usf`  
          `Engine/Shaders/Private/Lumen/LumenReflectionHardwareRayTracing.usf`

---

## LumenReflections.usf（タイル分類・レイ生成）

### InitReflectionIndirectArgsCS

```hlsl
[numthreads(1, 1, 1)]
void InitReflectionIndirectArgsCS()
```
| 目的 | タイル分類後の Indirect Dispatch 引数を初期化 |

### ReflectionTileClassificationBuildListsCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ReflectionTileClassificationBuildListsCS(uint3 GroupId, uint3 GroupThreadId)
```
| 項目 | 内容 |
|-----|------|
| **目的** | Roughness・ShadingModel でタイルを「トレースあり / なし / Radiance Cache のみ」に分類 |
| **入力** | `SceneDepthTexture`, `GBufferB`（Roughness）|
| **出力** | タイルリストバッファ（`RWReflectionTileData`）|
| **CPU 関数** | `ClassifyReflectionTiles()` |

### ReflectionGenerateRaysCS

```hlsl
[numthreads(REFLECTION_THREADGROUP_SIZE_1D, 1, 1)]
void ReflectionGenerateRaysCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 項目 | 内容 |
|-----|------|
| **目的** | BRDF インポータンスサンプリングでレイ方向・MIS 重みを計算 |
| **入力** | `SceneNormal`, `Roughness`, `FrameJitter` |
| **出力** | `RWRayBuffer`（float4: Direction.xyz, MaxDistance）, `RWRayTraceDistance`（uint パック）|
| **CPU 関数** | `GenerateReflectionRays()` |

### ReflectionClearUnusedTraceTileDataCS / ReflectionClearNeighborTileCS

```hlsl
[numthreads(REFLECTION_THREADGROUP_SIZE_1D, 1, 1)]
void ReflectionClearUnusedTraceTileDataCS(...)
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ReflectionClearNeighborTileCS(...)
```
| 目的 | 未使用タイルのトレースバッファをクリア（後続パスの誤参照防止）|

---

## LumenReflectionTracing.usf（トレース）

### ReflectionClearTracesCS

```hlsl
[numthreads(REFLECTION_THREADGROUP_SIZE_1D, 1, 1)]
void ReflectionClearTracesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | `TraceHit` / `TraceRadiance` バッファの初期化 |

### ReflectionTraceScreenTexturesCS

```hlsl
[numthreads(REFLECTION_THREADGROUP_SIZE_1D, 1, 1)]
void ReflectionTraceScreenTexturesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | HZB を使ったスクリーンスペーストレース（近距離・高精度）|
| 出力 | ヒット情報を `RWTraceHit`（float）に記録 |

### ReflectionCompactTracesCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void ReflectionCompactTracesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | Screen Trace ミスのレイのみを Compact（後続 Mesh SDF トレースの対象を絞る）|

### ReflectionSortTracesByMaterialCS

```hlsl
[numthreads(THREADGROUP_SIZE_1D, 1, 1)]
void ReflectionSortTracesByMaterialCS(uint DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | Material ID でレイをソートしてキャッシュ効率を向上（Radiance Cache と同様の手法）|

### ReflectionTraceMeshSDFsCS

```hlsl
[numthreads(REFLECTION_TRACE_THREADGROUP_SIZE_1D, 1, 1)]
void ReflectionTraceMeshSDFsCS(uint DispatchThreadId : SV_DispatchThreadID)
```
| 項目 | 内容 |
|-----|------|
| **目的** | Screen Trace ミスのレイを Mesh SDF でトレース |
| **内部** | `TraceMeshSDFs()` inline 関数 → `LumenSoftwareRayTracing.ush` |
| **出力** | `RWTraceHit`（ヒット距離 float）, `RWTraceRadiance`（float3 輝度）|

### ReflectionTraceVoxelsCS

```hlsl
[numthreads(REFLECTION_TRACE_THREADGROUP_SIZE_1D, 1, 1)]
void ReflectionTraceVoxelsCS(uint DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | Global SDF（Voxel）でトレース（Mesh SDF ミス後）|

---

## LumenReflectionResolve.usf

### LumenReflectionResolveCS

```hlsl
[numthreads(REFLECTION_THREADGROUP_SIZE_2D, REFLECTION_THREADGROUP_SIZE_2D, 1)]
void LumenReflectionResolveCS(uint3 GroupId, uint2 GroupThreadId)
```
| 項目 | 内容 |
|-----|------|
| **目的** | ダウンサンプルされたトレース結果を近傍ピクセルのデプス・法線で重み付けして展開 |
| **入力** | `TraceRadiance`（ダウンサンプル）, `SceneDepth`, `SceneNormal` |
| **出力** | `RWSpecularIndirect`（float3）, `RWSpecularIndirectDepth`（float）|
| **CPU 関数** | `ResolveReflections()` |

---

## LumenReflectionHardwareRayTracing.usf

```hlsl
RAY_TRACING_ENTRY_RAYGEN(LumenReflectionHardwareRayTracingRGS)
```
| 目的 | HW RT で反射レイをトレース（`r.Lumen.Reflections.HardwareRayTracing=1` 時）|
| CPU 関数 | `DispatchLumenReflectionHardwareRayTracing()` |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Lumen.Reflections.MaxRoughnessToTrace` | 0.4 | トレースする最大 Roughness |
| `r.Lumen.Reflections.DownsampleFactor` | 1 | レイのダウンサンプル率（1=フル解像度）|
| `r.Lumen.Reflections.HardwareRayTracing` | 0 | HW RT 反射を使うか |
| `r.Lumen.Reflections.TemporalMaxFramesAccumulated` | 16 | テンポラル蓄積上限 |
| `r.Lumen.Reflections.ScreenSpaceReconstruction` | 1 | スクリーンスペース再構成有効 |

---

> [!note]- ダウンサンプルとResolveの品質トレードオフ
> `r.Lumen.Reflections.DownsampleFactor` を 2 にすると反射レイを半解像度でトレースし、`LumenReflectionResolveCS()` で元解像度に展開する。  
> コストは約 1/4 になるが、近傍ピクセルの法線・深度が大きく異なる境界部（エッジ、段差）でアーティファクトが出やすくなる。  
> デフォルトの 1（フル解像度）は最高品質だが、Roughness の小さい鏡面マテリアルが多い場面では GPU 負荷の主要因になる。
