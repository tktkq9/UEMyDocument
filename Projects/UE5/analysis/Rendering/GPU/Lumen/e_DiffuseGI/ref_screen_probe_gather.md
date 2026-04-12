# REF: Lumen Screen Probe Gather シェーダー

- グループ: e - Diffuse GI
- 詳細: [[detail_screen_probe_gather]]
- CPU リファレンス: [[ref_lumen_diffuse_indirect]] | [[ref_lumen_screen_probe_gather]]
- ソース: `Engine/Shaders/Private/Lumen/LumenScreenProbeGather.usf`  
          `Engine/Shaders/Private/Lumen/LumenScreenProbeTracing.usf`  
          `Engine/Shaders/Private/Lumen/LumenScreenProbeGatherTemporal.usf`  
          `Engine/Shaders/Private/Lumen/LumenScreenProbeFiltering.usf`  
          `Engine/Shaders/Private/Lumen/LumenScreenProbeHardwareRayTracing.usf`

---

## LumenScreenProbeGather.usf

### ScreenProbeDownsampleDepthUniformCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ScreenProbeDownsampleDepthUniformCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | 8×8 タイルごとに代表深度・法線をダウンサンプルしてプローブ配置位置を決定 |
| 出力 | `RWScreenProbeSceneDepth`, `RWScreenProbeWorldNormal`, `RWScreenProbeTranslatedWorldPosition` |

### SetupAdaptiveProbeIndirectArgsCS

```hlsl
[numthreads(1, 1, 1)]
void SetupAdaptiveProbeIndirectArgsCS()
```
| 目的 | 適応的プローブ追加用の Indirect Dispatch 引数を設定 |

### MarkRadianceProbesUsedByScreenProbesCS

```hlsl
[numthreads(PROBE_THREADGROUP_SIZE_2D, PROBE_THREADGROUP_SIZE_2D, 1)]
void MarkRadianceProbesUsedByScreenProbesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | 各 Screen Probe の遠距離レイ方向に対応する Radiance Cache プローブをマーク |

### InitScreenProbeTileIndirectArgsCS

```hlsl
[numthreads(32, 1, 1)]
void InitScreenProbeTileIndirectArgsCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | タイル分類後の Indirect 引数を初期化 |

### ScreenProbeTileClassificationBuildListsCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void ScreenProbeTileClassificationBuildListsCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | 画面タイルをタイプ（均一プローブ / 適応的プローブ / 空タイル）に分類してリスト化 |

### ScreenProbeIntegrateCS

```hlsl
[numthreads(INTEGRATE_TILE_SIZE, INTEGRATE_TILE_SIZE, 1)]
void ScreenProbeIntegrateCS(uint3 GroupId, uint3 GroupThreadId)
```
| 項目 | 内容 |
|-----|------|
| **目的** | 各プローブのトレース結果を補間・双線形フィルタリングでスクリーン全ピクセルに展開 |
| **入力** | `TracedRadiance`（トレース結果）, `ScreenProbeWorldNormal`, `SceneDepth` |
| **出力** | `RWDiffuseIndirect`（Texture2DArray）, `RWRoughSpecularIndirect` |
| **CPU 関数** | `IntegrateScreenProbes()` |

### ScreenProbeAdaptivePlacementMarkCS / SpawnCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ScreenProbeAdaptivePlacementMarkCS(...)
void ScreenProbeAdaptivePlacementSpawnCS(...)
```
| 目的 | 輝度分散が大きいタイルに追加プローブを配置（適応的サンプリング）|

---

## LumenScreenProbeTracing.usf

### ClearTracesCS

```hlsl
[numthreads(PROBE_THREADGROUP_SIZE_2D, PROBE_THREADGROUP_SIZE_2D, 1)]
void ClearTracesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | トレースバッファ（TraceHit / TraceRadiance）を初期化 |

### ScreenProbeTraceScreenTexturesCS

```hlsl
[numthreads(PROBE_THREADGROUP_SIZE_2D, PROBE_THREADGROUP_SIZE_2D, 1)]
void ScreenProbeTraceScreenTexturesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | HZB（Hierarchical Z-Buffer）を使ったスクリーンスペーストレース（近距離・高精度）|
| 出力 | ヒットしたレイを `TraceHit` / `TraceRadiance` に記録 |

### ScreenProbeCompactTracesCS

```hlsl
[numthreads(PROBE_THREADGROUP_SIZE_2D, PROBE_THREADGROUP_SIZE_2D, 1)]
void ScreenProbeCompactTracesCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | Screen Trace でミスしたレイのみを Compact（後続トレースの対象を絞る）|

### ScreenProbeTraceMeshSDFsCS

```hlsl
[numthreads(PROBE_TRACE_THREADGROUP_SIZE_1D, 1, 1)]
void ScreenProbeTraceMeshSDFsCS(uint DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | Mesh SDF でトレース（Screen Trace ミスのレイを対象）|
| 内部 | `TraceMeshSDFs()` inline 関数を呼ぶ → `LumenSoftwareRayTracing.ush` |

### ScreenProbeTraceVoxelsCS

```hlsl
[numthreads(PROBE_TRACE_THREADGROUP_SIZE_1D, 1, 1)]
void ScreenProbeTraceVoxelsCS(uint DispatchThreadId : SV_DispatchThreadID)
```
| 目的 | Global SDF（Voxel）でトレース（Mesh SDF ミスのレイを対象）または Radiance Cache から補完 |

---

## LumenScreenProbeGatherTemporal.usf

### ScreenProbeTemporalReprojectionCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ScreenProbeTemporalReprojectionCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```
| 項目 | 内容 |
|-----|------|
| **目的** | 前フレームの GI 履歴（HistoryDiffuseIndirect）と現フレームをモーションベクターでリプロジェクトしてブレンド |
| **入力** | `DiffuseIndirect`（現フレーム）, `HistoryDiffuseIndirect`（前フレーム）, `SceneVelocity` |
| **出力** | `RWNewHistoryDiffuseIndirect`, `RWNewHistoryRoughSpecularIndirect`, `RWNewHistoryFastUpdateMode_NumFramesAccumulated` |
| **CPU 関数** | `TemporallyAccumulateScreenProbes()` |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Lumen.ScreenProbeGather.DownsampleFactor` | 16 | Screen Probe グリッドのダウンサンプル率（16→1/16 解像度）|
| `r.Lumen.ScreenProbeGather.TracingOctahedronResolution` | 8 | プローブの方向サンプリング解像度（8×8=64 方向）|
| `r.Lumen.ScreenProbeGather.ScreenTraces` | 1 | Screen Trace（HZB）有効化 |
| `r.Lumen.ScreenProbeGather.TemporalMaxFramesAccumulated` | 16 | テンポラル蓄積フレーム上限 |
| `r.Lumen.ScreenProbeGather.AdaptiveProbeMinDownsampleFactor` | 8 | 適応的プローブの最小グリッドサイズ |

---

> [!note]- Screen Probe の「補間」がフルピクセル GI を生成する仕組み
> Screen Probe は 8×8 タイルに 1つ（適応的なら複数）配置されるだけで、全ピクセルにはプローブがない。  
> `ScreenProbeIntegrateCS()` は各ピクセルに対して最近傍 4 プローブの輝度を深度・法線で重み付きして補間する。  
> 補間に使う重みは「プローブの法線とピクセルの法線の内積」「深度の差」で決まり、表面が異なる物体のプローブが混ざらないようになっている。

> [!note]- Fast Update Mode と NumFramesAccumulated
> テンポラル蓄積では通常 `1/N` のブレンドで安定化するが、高速移動・ライト変化時は Ghost を避けるため `FastUpdateMode` で蓄積量を増やす。  
> `RWNewHistoryFastUpdateMode_NumFramesAccumulated` は「どのピクセルが急激に変化したか」をフラグとフレーム数で記録し、`ScreenProbeTemporalReprojectionCS()` がピクセルごとにブレンドアルファを変えて処理する。
