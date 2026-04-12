# Lumen Reflections シェーダー詳細

- グループ: f - Reflections
- GPU 概要: [[01_lumen_gpu_overview]]
- CPU 詳細: [[f_lumen_reflections]]
- リファレンス: [[ref_reflection_tracing]] | [[ref_reflection_denoiser]]

---

## 概要

Lumen の反射計算パス。Roughness に応じてトレースの精度・コストを動的に切り替え、  
**Resolve（再構成）→ デノイズ（テンポラル/スペーシャル）** の順で最終反射テクスチャを生成する。

---

## パス構成

```
[1] タイル分類（ReflectionTileClassificationBuildListsCS）
     → スクリーンを Roughness + 遮蔽で分類してタイルリスト生成

[2] レイ生成（ReflectionGenerateRaysCS）
     → 各ピクセルのレイ方向・MIS 重みを計算 → RWRayBuffer

[3] トレース（LumenReflectionTracing.usf）
     ├─ Screen Trace（ReflectionTraceScreenTexturesCS）
     ├─ Mesh SDF（ReflectionTraceMeshSDFsCS）
     └─ Global SDF（ReflectionTraceVoxelsCS）

[4] Resolve（LumenReflectionResolve.usf#LumenReflectionResolveCS）
     → ダウンサンプル反射レイをピクセル解像度に再構成

[5] デノイズ
     ├─ Temporal（LumenReflectionDenoiserTemporal.usf#LumenReflectionDenoiserTemporalCS）
     └─ Spatial（LumenReflectionDenoiserSpatial.usf#LumenReflectionDenoiserSpatialCS）

     ↓
FSSDSignalTextures::Textures[3]（反射バッファ）
```

---

## 主要シェーダーロジック

### [1] タイル分類（LumenReflections.usf）

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ReflectionTileClassificationBuildListsCS(...)
```

各ピクセルの Roughness と ShadingModel に基づいてタイルを分類：
- **Mirror Roughness**（≈0.0）: 高精度トレース
- **Glossy**（0.0〜MaxRoughnessToTrace）: 通常トレース
- **Diffuse**（>MaxRoughnessToTrace）: Radiance Cache から補完（トレースしない）

### [2] レイ生成（LumenReflections.usf）

```hlsl
[numthreads(REFLECTION_THREADGROUP_SIZE_1D, 1, 1)]
void ReflectionGenerateRaysCS(...)
```

- `RWRayBuffer`：レイ方向（float4）とレイ距離をピクセルごとに書き込み
- インポータンスサンプリングで BRDF に基づく方向サンプリング

### [3] Mesh SDF トレース（LumenReflectionTracing.usf）

```hlsl
[numthreads(REFLECTION_TRACE_THREADGROUP_SIZE_1D, 1, 1)]
void ReflectionTraceMeshSDFsCS(uint DispatchThreadId : SV_DispatchThreadID)
void ReflectionTraceVoxelsCS(uint DispatchThreadId : SV_DispatchThreadID)
```

- Screen Probe と同様の3段階トレース（Screen → Mesh SDF → Global SDF）
- ヒット輝度を `RWTraceRadiance`（float3）に書き込み

### [4] Resolve（LumenReflectionResolve.usf）

```hlsl
[numthreads(REFLECTION_THREADGROUP_SIZE_2D, REFLECTION_THREADGROUP_SIZE_2D, 1)]
void LumenReflectionResolveCS(...)
```

- ダウンサンプルされた反射レイ結果を近傍ピクセルに展開・再構成
- 出力: `RWSpecularIndirect`（最終反射輝度）, `RWSpecularIndirectDepth`

### [5] テンポラルデノイズ（LumenReflectionDenoiserTemporal.usf）

```hlsl
[numthreads(REFLECTION_THREADGROUP_SIZE_2D, REFLECTION_THREADGROUP_SIZE_2D, 1)]
void LumenReflectionDenoiserTemporalCS(...)
```

- モーションベクターで前フレームの反射履歴をリプロジェクト
- 輝度の 2nd モーメントで適応的なブレンドアルファを決定
- 出力: `RWSpecularAndSecondMoment`（float4：反射 RGB + 2次モーメント）, `RWNumFramesAccumulated`

### [6] スペーシャルデノイズ（LumenReflectionDenoiserSpatial.usf）

```hlsl
[numthreads(REFLECTION_THREADGROUP_SIZE_2D, REFLECTION_THREADGROUP_SIZE_2D, 1)]
void LumenReflectionDenoiserSpatialCS(...)
```

- 隣接ピクセルをバイラテラルフィルタで統合（法線・深度・Roughness で重み付け）
- 出力: `RWSpecularIndirectAccumulated`（テンポラル + スペーシャル統合済み）

---

## 入出力

### 入力

| リソース | 内容 |
|---------|------|
| `SceneDepth`, `SceneNormal`, `SceneRoughness` | GBuffer からの Roughness 等 |
| `FinalLightingAtlas` | SDF ヒット点の輝度サンプリング元 |
| `RadianceCacheParameters` | Roughness 大のピクセルへの Radiance Cache 補完 |

### 出力

| テクスチャ | 内容 |
|---------|------|
| `RWRayBuffer` | レイ方向・距離（中間）|
| `RWTraceRadiance` | トレース輝度（中間）|
| `RWSpecularIndirect` | Resolve 後の反射輝度 |
| `RWSpecularIndirectAccumulated` | デノイズ済み最終反射（→ DiffuseIndirectComposite へ）|

---

## CPU 呼び出しの流れ

```
RenderLumenReflections()                      // LumenReflections.cpp
  │
  ├─ ClassifyReflectionTiles()                → ReflectionTileClassificationBuildListsCS
  ├─ GenerateReflectionRays()                 → ReflectionGenerateRaysCS
  ├─ TraceReflections()
  │   ├─ ReflectionTraceScreenTexturesCS
  │   ├─ ReflectionCompactTracesCS
  │   ├─ ReflectionTraceMeshSDFsCS
  │   └─ ReflectionTraceVoxelsCS
  ├─ [HW RT 時] LumenReflectionHardwareRayTracing.usf
  ├─ ResolveReflections()                     → LumenReflectionResolveCS
  ├─ DenoiseReflections_Temporal()            → LumenReflectionDenoiserTemporalCS
  └─ DenoiseReflections_Spatial()             → LumenReflectionDenoiserSpatialCS
       ↓
  返値: FRDGTexture*（SpecularIndirectAccumulated → Textures[3] として FinalComposite へ）
```
