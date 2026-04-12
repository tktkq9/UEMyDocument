# REF: Lumen Radiosity シェーダー

- グループ: b - Scene Lighting
- 詳細: [[detail_radiosity]]
- CPU リファレンス: [[ref_lumen_radiosity]]
- ソース: `Engine/Shaders/Private/Lumen/Radiosity/LumenRadiosity.usf`  
          `Engine/Shaders/Private/Lumen/Radiosity/LumenRadiosityCulling.usf`  
          `Engine/Shaders/Private/Lumen/Radiosity/LumenRadiosityHardwareRayTracing.usf`

---

## LumenRadiosity.usf

### LumenRadiosityIndirectArgsCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void LumenRadiosityIndirectArgsCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Indirect Dispatch の引数バッファを設定（更新する Probe 数を確定）|
| **CPU 関数** | `RenderRadiosityForLumenScene()` の前処理 |

### LumenRadiosityDistanceFieldTracingCS

```hlsl
[numthreads(RADIOSITY_TRACE_THREADGROUP_SIZE, 1, 1)]
void LumenRadiosityDistanceFieldTracingCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 各 Radiosity Probe からレイを SDF でトレースして輝度を収集 |
| **入力** | `GlobalSDF`, `LumenCardData`（FinalLightingAtlas 参照のため）|
| **出力** | `RWTraceRadianceAtlas`（float3 輝度）, `RWTraceHitDistanceAtlas`（float 距離）|
| **CPU 関数** | `DispatchRadiosityTracing()` in `LumenRadiosity.cpp` |

#### パーミュテーション

| クラス | 説明 |
|-------|------|
| `TRACE_GLOBAL_SDF` | Global SDF でトレース（Mesh SDF カバー外）|
| `INDIRECT_DISPATCH` | Indirect Dispatch 対応 |

### LumenRadiositySpatialFilterProbeRadiance

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void LumenRadiositySpatialFilterProbeRadiance(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Probe 輝度のスペーシャルフィルタリング（隣接 Probe との加重平均）|
| **入力** | `TraceRadianceAtlas`, `TraceHitDistanceAtlas` |
| **出力** | `RWFilteredTraceRadianceAtlas` |

### SampleTraceRadianceAtlas（ユーティリティ）

```hlsl
float3 SampleTraceRadianceAtlas(
    uint2 ProbeCoord,
    uint  RayIndex,
    float3 Direction)
```

| 目的 | フィルタリング後の Probe 輝度をサンプリング（ConvertToSH で使用）|
|-----|------|

### LumenRadiosityConvertToSH

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void LumenRadiosityConvertToSH(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Probe 輝度（半球サンプル群）を L1 SH（4係数 × RGB = 12 float）に変換 |
| **出力** | `RWRadiosityProbeSHRedAtlas` / `RWRadiosityProbeSHGreenAtlas` / `RWRadiosityProbeSHBlueAtlas` |

### SampleRadiositySH（ユーティリティ）

```hlsl
float3 SampleRadiositySH(uint2 ProbeAtlasCoord, float3 Direction)
```

| 目的 | SH 係数から任意方向への Irradiance を計算（IntegrateCS で使用）|
|-----|------|

### LumenRadiosityIntegrateCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void LumenRadiosityIntegrateCS(
    uint3 GroupId       : SV_GroupID,
    uint3 GroupThreadId : SV_GroupThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | SH 係数をカードテクセルの法線方向で積分し Irradiance を生成。テンポラル蓄積で安定化 |
| **入力** | SH アトラス 3枚（Red/Green/Blue）, `RadiosityNumFramesAccumulatedAtlas` |
| **出力** | `RWRadiosityAtlas`（float3 間接光 Irradiance）, `RWRadiosityNumFramesAccumulatedAtlas`（蓄積フレーム数）|
| **CPU 関数** | `IntegrateRadiosity()` in `LumenRadiosity.cpp` |
| **テンポラル式** | `Output = lerp(OldRadiosity, NewIrradiance, BlendAlpha)`  `BlendAlpha = 1 / (NumFramesAccumulated + 1)` |

---

## LumenRadiosityCulling.usf

### 主要エントリポイント（詳細はソース参照）

| エントリポイント | 役割 |
|---------------|------|
| `InitRadiosityProbeIndirectArgsCS()` | Probe カリング用 Indirect 引数初期化 |
| `BuildRadiosityProbesCS()` | 更新対象 Probe リストを構築 |
| `SplatCardPageRadiosityCS()` | CardPage を Probe グリッドにスプラット |

---

## LumenRadiosityHardwareRayTracing.usf

```hlsl
RAY_TRACING_ENTRY_RAYGEN(LumenRadiosityHardwareRayTracingRGS)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Ray Generation Shader（DXR）|
| **目的** | HW Ray Tracing で Radiosity Probe のレイトレース（`r.Lumen.Radiosity.HardwareRayTracing=1` 時）|
| **出力** | SW RT 版と同じ `RWTraceRadianceAtlas` / `RWTraceHitDistanceAtlas` |
| **CPU 関数** | `DispatchRadiosityHardwareRayTracing()` |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Lumen.Radiosity.ProbeSpacingInWorldSpace` | 100 | Probe 間隔（ワールド単位）|
| `r.Lumen.Radiosity.NumRaysPerProbe` | 8 | 1プローブあたりのレイ数 |
| `r.Lumen.Radiosity.TemporalMaxFramesAccumulated` | 8 | テンポラル蓄積の最大フレーム数 |
| `r.Lumen.Radiosity.HardwareRayTracing` | 0 | HW RT 版を使うか |
| `r.Lumen.Radiosity.Allow` | 1 | Radiosity パスの有効化 |

---

> [!note]- Radiosity が担う「1バウンス間接光」の役割
> Lumen の Radiosity は Surface Cache 上の「テクセル視点からの反射光」を計算し、その結果を `RadiosityAtlas` に格納する。  
> Screen Probe Gather や Radiance Cache がシーンをトレースしてサーフェスにヒットしたとき、`FinalLightingAtlas`（Direct + Radiosity の合成）をサンプリングして輝度を得る。  
> つまり Radiosity は「サーフェスがどれだけ明るく見えるか」を Surface Cache に事前計算する担当。

> [!note]- SH 表現と Probe ごとの 8 レイ
> 1 Probe あたり 8 方向のレイを打つことで半球の輝度分布を粗くサンプリングし、これを L1 SH（4係数）に変換する。  
> L1 SH は方向依存性を低コストで表現でき、どの法線方向への Irradiance も 1回の内積で計算できる。  
> 品質を上げるには `r.Lumen.Radiosity.NumRaysPerProbe` を増やすが、トレースコストが比例して増加する。

> [!note]- テンポラル蓄積と Ghost アーティファクト
> `LumenRadiosityIntegrateCS` は `NumFramesAccumulated` を毎フレーム +1 してブレンドアルファを減衰させることで、フレームをまたぐ安定した Irradiance を実現する。  
> ライトが急変したり Card が移動した場合は蓄積をリセットしてアルファ = 1 から再蓄積を始める（Ghost アーティファクト防止）。  
> `r.Lumen.Radiosity.TemporalMaxFramesAccumulated` で蓄積上限を制限でき、過去のライティングが長く残らないようにする。
