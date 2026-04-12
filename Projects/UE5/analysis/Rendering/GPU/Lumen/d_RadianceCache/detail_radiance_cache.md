# Lumen Radiance Cache シェーダー詳細

- グループ: d - Radiance Cache
- GPU 概要: [[01_lumen_gpu_overview]]
- CPU 詳細: [[d_lumen_radiance_cache]]
- リファレンス: [[ref_radiance_cache]]

---

## 概要

**Radiance Cache** は 3D 空間に疎に配置したプローブキャッシュ。  
Screen Probe Gather が届かない遠距離・カメラ外領域の間接光を低コストで補う。  
各プローブは「その位置から見える輝度を全方向に記録したアトラステクスチャ」として管理される。

> Surface Cache が「サーフェス上の照明」を記録するのに対し、  
> Radiance Cache は「空間上の任意点での輝度」を球面的に記録する。

---

## プローブのライフサイクル

```
[1] AllocateUsedProbesCS()     — 使用中プローブの確定
[2] SelectMaxPriorityBucketCS() — 更新バジェット決定
[3] AllocateProbeTracesCS()    — 更新するプローブのトレース割り当て
[4] UpdateCacheForUsedProbesCS() — 新しいプローブの配置
    ↓
[5] GenerateProbeTraceTilesCS() — プローブのトレースタイルを生成
[6] SortProbeTraceTilesCS()     — GPU ソートで Material ID でグループ化
[7] TraceFromProbesCS()         — SDF / HW RT でレイをトレース
    ↓
[8] FilterProbeRadianceWithGatherCS() — フィルタリング
[9] CalculateProbeIrradianceCS()      — 輝度 → Irradiance 変換
[10] FixupBordersAndGenerateMipsCS()  — ミップ生成
```

---

## 主要パス

### TraceFromProbesCS（コアトレース）

```hlsl
[numthreads(RADIANCE_CACHE_TRACE_TILE_SIZE_2D, RADIANCE_CACHE_TRACE_TILE_SIZE_2D, 1)]
void TraceFromProbesCS(uint3 GroupId, uint3 GroupThreadId)
```

- プローブごとの球面タイルからレイを射出
- SDF または HW RT でトレース → FinalLightingAtlas をサンプル
- 出力: `RWRadianceProbeAtlasTexture`（プローブごとの輝度アトラス）

### CalculateProbeIrradianceCS（Irradiance 変換）

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void CalculateProbeIrradianceCS(uint3 GroupId, uint3 GroupThreadId)
```

- Probe の輝度アトラスを積分してコサイン加重 Irradiance に変換
- 出力: `RWFinalIrradianceAtlas`（Screen Probe Gather が補間に使用）

### FixupBordersAndGenerateMipsCS（ミップ生成）

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void FixupBordersAndGenerateMipsCS(uint3 GroupId, uint3 GroupThreadId)
```

- プローブアトラスの境界ダイレーション + ミップチェーン生成
- 出力: `RWFinalRadianceAtlasMip0/1/2`（補間に使うミップ付きアトラス）

---

## 入出力

### 入力

| リソース | 内容 |
|---------|------|
| `FinalLightingAtlas` | Surface Cache の最終照明（レイヒット時のサンプリング源）|
| `GlobalSDF` / TLAS | トレース用（SW / HW RT）|
| `RadianceCacheIndirectionTexture` | 空間インデックス → プローブアトラス座標の変換テーブル |

### 出力

| テクスチャ | 内容 |
|---------|------|
| `RWRadianceProbeAtlasTexture` | プローブごとの全方向輝度（中間）|
| `RWFinalIrradianceAtlas` | Irradiance 変換済み（Screen Probe が参照）|
| `RWFinalRadianceAtlasMip0/1/2` | ミップ付き最終輝度アトラス |
| `RWRadianceCacheProbeOcclusionAtlas` | プローブ遮蔽マスク |

---

## CPU 呼び出しの流れ

```
LumenRadianceCache::Render()          // LumenRadianceCache.cpp
  │
  ├─ LumenRadianceCacheUpdate.usf 系   ← プローブ割り当て・更新リスト生成
  ├─ LumenRadianceCache.usf#GenerateProbeTraceTilesCS
  ├─ LumenRadianceCache.usf#SortProbeTraceTilesCS
  ├─ LumenRadianceCache.usf#TraceFromProbesCS          ← コアトレース（SW/HW RT）
  ├─ [HW RT 時] LumenRadianceCacheHardwareRayTracing.usf
  ├─ LumenRadianceCache.usf#FilterProbeRadianceWithGatherCS
  ├─ LumenRadianceCache.usf#CalculateProbeIrradianceCS
  └─ LumenRadianceCache.usf#FixupBordersAndGenerateMipsCS
```

**Screen Probe との接続：**

```cpp
// LumenScreenProbeGather.cpp 内
// Radiance Cache の補間パラメーターを Screen Probe トレース CS に渡す
ShaderParameters.RadianceCacheParameters = RadianceCacheInterpolationParameters;
// → LumenRadianceCacheInterpolation.ush#SampleRadianceCacheInterpolated() で使用
```
