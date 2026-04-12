# Lumen Radiosity シェーダー詳細

- グループ: b - Scene Lighting
- GPU 概要: [[01_lumen_gpu_overview]]
- CPU 詳細: [[b_lumen_scene_lighting]]
- リファレンス: [[ref_radiosity]]

---

## 概要

Surface Cache に **間接光（Radiosity）** を焼き込むパス。  
シーン内の各サーフェス（Card）から FinalLightingAtlas にレイをトレースし、  
その輝度を SH（球面調和関数）に変換してテンポラル蓄積する。

> GBuffer の AO とは異なり、Surface Cache 上のテクセル視点でのバウンスライトを計算する。  
> これは Lumen の「1バウンスの間接光」の主要な計算源。

---

## パス構成

```
RenderRadiosityForLumenScene()         // LumenRadiosity.cpp
  │
  ├─ [1] LumenRadiosityCulling.usf
  │       → Probe の更新リストを生成・Indirect Dispatch 引数を設定
  │
  ├─ [2] LumenRadiosity.usf#LumenRadiosityDistanceFieldTracingCS
  │       → 各 Probe からレイをトレース（SDF または HW RT）
  │       → RWTraceRadianceAtlas にヒット輝度を書き込み
  │
  ├─ [2b] LumenRadiosityHardwareRayTracing.usf（HW RT 時）
  │        → DXR で Probe レイをトレース
  │
  ├─ [3] LumenRadiosity.usf#LumenRadiositySpatialFilterProbeRadiance
  │       → Probe 輝度のスペーシャルフィルタリング
  │
  ├─ [4] LumenRadiosity.usf#LumenRadiosityConvertToSH
  │       → Probe 輝度を SH（L1 = 4係数 × RGB）に変換
  │
  └─ [5] LumenRadiosity.usf#LumenRadiosityIntegrateCS
          → SH を積分してカードの Irradiance を生成
          → RWRadiosityAtlas に書き込み（テンポラル蓄積あり）
```

---

## Radiosity Probe の配置

各 Card は 2D グリッドに **Radiosity Probe** を配置する。  
プローブごとに複数のレイを飛ばして輝度を収集する。

```
Card (Surface Cache の 1テクスチャ領域)
  └─ 8×8 = 64 Probe（RADIOSITY_PROBE_RESOLUTION のデフォルト）
       └─ 各 Probe から 8方向（半球）にレイを射出
            └─ SDF / HW RT でトレース → FinalLightingAtlas をサンプル
```

---

## 主要シェーダーロジック

### LumenRadiosityDistanceFieldTracingCS

```hlsl
[numthreads(RADIOSITY_TRACE_THREADGROUP_SIZE, 1, 1)]
void LumenRadiosityDistanceFieldTracingCS(
    uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // 1. ProbeIndex から CardPage・CardUV を解決
    // 2. ProbeIndex と RayIndex からトレース方向を決定（半球上）
    // 3. SDF テクスチャでトレース（LumenSoftwareRayTracing.ush）
    // 4. ヒットした場合 FinalLightingAtlas をサンプリング
    // 5. RWTraceRadianceAtlas[ProbeAtlasCoord] = HitRadiance
    //    RWTraceHitDistanceAtlas[ProbeAtlasCoord] = HitDistance
}
```

### LumenRadiosityIntegrateCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void LumenRadiosityIntegrateCS(
    uint3 GroupId : SV_GroupID,
    uint3 GroupThreadId : SV_GroupThreadID)
{
    // 1. ProbeRadiance（FilteredTraceRadianceAtlas）から SH を取得
    // 2. SH を使ってカードのテクセルへの Irradiance を計算
    // 3. テンポラル蓄積（線形補間）で RadiosityAtlas を更新
    //    RWRadiosityAtlas[AtlasCoord] = lerp(OldValue, NewIrradiance, BlendAlpha)
    //    RWRadiosityNumFramesAccumulatedAtlas[AtlasCoord] = FrameCount
}
```

---

## 入出力

### 入力

| リソース | 内容 |
|---------|------|
| `FinalLightingAtlas` | レイヒット時のサンプリング元（Direct + Emissive）|
| `AlbedoAtlas` | Radiosity ヒットサーフェスの Albedo（次バウンス Albedo 係数）|
| `GlobalSDF` / `MeshSDF` | SW RT 時のトレース用 SDF テクスチャ |

### 出力

| テクスチャ | 内容 |
|---------|------|
| `RWTraceRadianceAtlas` | Probe ごとのトレース輝度（中間バッファ）|
| `RWTraceHitDistanceAtlas` | Probe ごとのヒット距離（中間バッファ）|
| `RWRadiosityAtlas` | テンポラル蓄積済み間接光 Irradiance（最終出力）|
| `RWRadiosityProbeSHRedAtlas` / `Green` / `Blue` | Probe の SH 係数（L1）|

---

## CPU 呼び出しの流れ

```
RenderLumenSceneLighting()                  // LumenSceneLighting.cpp:217
  │
  └─ RenderRadiosityForLumenScene()          // LumenRadiosity.cpp
       │
       ├─ LumenRadiosityCulling.usf          ← Probe 更新リスト生成
       ├─ LumenRadiosity.usf#LumenRadiosityDistanceFieldTracingCS  ← トレース
       ├─ [HW RT 時] LumenRadiosityHardwareRayTracing.usf
       ├─ LumenRadiosity.usf#LumenRadiositySpatialFilterProbeRadiance ← フィルタ
       ├─ LumenRadiosity.usf#LumenRadiosityConvertToSH               ← SH 変換
       └─ LumenRadiosity.usf#LumenRadiosityIntegrateCS               ← 積分 + テンポラル
```

**CPU 側パラメーター割り当て（概略）：**

```cpp
// LumenRadiosity.cpp 内
FLumenRadiosityDistanceFieldTracingCS::FParameters* PassParams = ...;
PassParams->RWTraceRadianceAtlas = GraphBuilder.CreateUAV(TraceRadianceAtlas);
PassParams->FinalLightingAtlas   = FinalLightingAtlas;  // SRV
PassParams->LumenSceneData       = GetLumenSceneData(GraphBuilder, View);
// → GraphBuilder.AddPass(…, PassParams, …)
```
