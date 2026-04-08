# リファレンス：LumenReflections.h / LumenReflections.cpp

- グループ: f - Reflections
- 上位: [[f_lumen_reflections]]
- 関連: [[ref_lumen_reflection_tracing]] | [[ref_lumen_reflection_hwrt]] | [[ref_lumen_radiance_cache]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenReflections.h/cpp`

---

## 概要

Lumen Reflections システムの公開 API と主要パラメータ構造体を定義するヘッダ・実装ファイル。  
反射レイのトレーシング設定・タイル分類・コンパクト化・合成パラメータを管理する。  
SW RT / HW RT の両方に対応し、Radiance Cache との連携も担う。

---

## LumenReflections 名前空間

### FCompositeParameters

反射の合成（最終結合）で使うパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FCompositeParameters, )
    SHADER_PARAMETER(float, MaxRoughnessToTrace)              // 反射レイをトレースする最大ラフネス
    SHADER_PARAMETER(float, MaxRoughnessToTraceForFoliage)    // フォリッジ用最大ラフネス
    SHADER_PARAMETER(float, InvRoughnessFadeLength)           // フェード幅の逆数
    SHADER_PARAMETER(float, ReflectionSmoothBias)             // スムース面へのバイアス
END_SHADER_PARAMETER_STRUCT()
```

### 判定・ユーティリティ関数

| 関数 | 説明 |
|-----|------|
| `SetupCompositeParameters(View, ReflectionsMethod, OutParameters)` | 合成パラメータをセットアップ |
| `UseAsyncCompute(ViewFamily, DiffuseIndirectMethod, ReflectionsMethod)` | AsyncCompute を使うか |
| `UseFarField(ViewFamily)` | Far Field を Reflections に使うか |
| `UseHitLighting(View, DiffuseIndirectMethod)` | Hit Lighting を使うか |
| `UseTranslucentRayTracing(View)` | 透明マテリアルの RT を使うか |
| `IsHitLightingForceEnabled(View, DiffuseIndirectMethod)` | Hit Lighting を強制するか |
| `UseSurfaceCacheFeedback()` | Surface Cache フィードバックを使うか |
| `UseScreenTraces(View)` | スクリーンスペーストレースを使うか |
| `UseDistantScreenTraces(View, bUseFarField, bUseRadianceCache)` | 遠距離スクリーントレースを使うか |
| `GetDistantScreenTraceStepOffsetBias()` | 遠距離トレースのステップオフセットバイアス |
| `UseRadianceCache()` | Radiance Cache を Reflections に使うか |
| `UseRadianceCacheSkyVisibility()` | Sky Visibility を Radiance Cache で計算するか |
| `UseRadianceCacheStochasticInterpolation()` | 確率的プローブ補間を使うか |
| `GetSampleSceneColorDepthTreshold()` | シーンカラーサンプリングの深度閾値 |
| `GetSampleSceneColorNormalTreshold()` | シーンカラーサンプリングの法線閾値 |
| `GetMaxReflectionBounces(View)` | 最大反射バウンス数 |
| `GetMaxRefractionBounces(View)` | 最大屈折バウンス数 |

---

## FLumenReflectionTracingParameters

反射トレーシング全体で共有するシェーダーパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenReflectionTracingParameters, )
    // デバッグ
    SHADER_PARAMETER_STRUCT_INCLUDE(FLumenReflectionsVisualizeTracesParameters, VisualizeTracesParameters)

    // ビューポート設定
    SHADER_PARAMETER(FIntPoint, ReflectionDownsampleFactorXY)    // ダウンサンプル係数
    SHADER_PARAMETER(FIntPoint, ReflectionTracingViewMin)
    SHADER_PARAMETER(FIntPoint, ReflectionTracingViewSize)
    SHADER_PARAMETER(FIntPoint, ReflectionTracingBufferSize)
    SHADER_PARAMETER(FVector2f, ReflectionTracingBufferInvSize)

    // トレース設定
    SHADER_PARAMETER(float,  MaxRayIntensity)         // レイ輝度クランプ
    SHADER_PARAMETER(uint32, ReflectionPass)           // パス番号（マルチバウンス用）
    SHADER_PARAMETER(uint32, UseJitter)                // ジッターを使うか
    SHADER_PARAMETER(uint32, UseHighResSurface)        // 高解像度サーフェスを使うか
    SHADER_PARAMETER(uint32, MaxReflectionBounces)     // 最大反射バウンス数
    SHADER_PARAMETER(uint32, MaxRefractionBounces)     // 最大屈折バウンス数
    SHADER_PARAMETER(uint32, ReflectionsStateFrameIndex)
    SHADER_PARAMETER(uint32, ReflectionsStateFrameIndexMod8)
    SHADER_PARAMETER(uint32, ReflectionsRayDirectionFrameIndex)

    // トレース距離
    SHADER_PARAMETER(float, NearFieldMaxTraceDistance)
    SHADER_PARAMETER(float, NearFieldMaxTraceDistanceDitherScale)
    SHADER_PARAMETER(float, NearFieldSceneRadius)
    SHADER_PARAMETER(float, FarFieldMaxTraceDistance)

    // 合成設定
    SHADER_PARAMETER_STRUCT_INCLUDE(LumenReflections::FCompositeParameters, ReflectionsCompositeParameters)
    SHADER_PARAMETER_TEXTURE(Texture2D, PreIntegratedGF)
    SHADER_PARAMETER_SAMPLER(SamplerState, PreIntegratedGFSampler)

    // レイバッファ
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float4>, RayBuffer)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<uint>, RayTraceDistance)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DownsampledDepth)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DownsampledClosureIndex)

    // トレース結果（SRV）
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, TraceHit)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, TraceRadiance)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, TraceMaterialId)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, TraceBookmark)

    // トレース結果（UAV）
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2DArray<float3>, RWTraceRadiance)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2DArray<float3>, RWTraceBackgroundVisibility)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2DArray<float>,  RWTraceHit)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2DArray<uint>,   RWTraceMaterialId)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2DArray<uint2>,  RWTraceBookmark)

    SHADER_PARAMETER_STRUCT_REF(FBlueNoise, BlueNoise)
END_SHADER_PARAMETER_STRUCT()
```

---

## FLumenReflectionTileParameters / FCompactedReflectionTraceParameters

```cpp
// タイルベースの反射処理用パラメータ
BEGIN_SHADER_PARAMETER_STRUCT(FLumenReflectionTileParameters, )
    SHADER_PARAMETER_RDG_TEXTURE(Texture2DArray<uint>, LumenTileBitmask)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, ReflectionClearTileData)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, ReflectionResolveTileData)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, ReflectionTracingTileData)
    RDG_BUFFER_ACCESS(ClearIndirectArgs, ERHIAccess::IndirectArgs)
    RDG_BUFFER_ACCESS(ResolveIndirectArgs, ERHIAccess::IndirectArgs)
    RDG_BUFFER_ACCESS(TracingIndirectArgs, ERHIAccess::IndirectArgs)
END_SHADER_PARAMETER_STRUCT()

// コンパクト化されたトレースパラメータ
BEGIN_SHADER_PARAMETER_STRUCT(FCompactedReflectionTraceParameters, )
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, CompactedTraceTexelAllocator)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, CompactedTraceTexelData)
    RDG_BUFFER_ACCESS(IndirectArgs, ERHIAccess::IndirectArgs)
    RDG_BUFFER_ACCESS(RayTraceDispatchIndirectArgs, ...)
END_SHADER_PARAMETER_STRUCT()
```

---

## ETraceCompactionMode / CompactTraces

```cpp
enum ETraceCompactionMode { Default, FarField, HitLighting, MAX };

// 反射トレースのコンパクト化（不要なピクセルを除外してカウンタバッファに積む）
FCompactedReflectionTraceParameters LumenReflections::CompactTraces(
    FRDGBuilder&, const FViewInfo&,
    const FLumenCardTracingParameters&,
    const FLumenReflectionTracingParameters&,
    const FLumenReflectionTileParameters&,
    bool bCullByDistanceFromCamera,
    float CompactionTracingEndDistanceFromCamera,
    float CompactionMaxTraceDistance,
    ERDGPassFlags,
    ETraceCompactionMode TraceCompactionMode,
    bool bSortByMaterial);
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.Reflections.Allow` | 1 | Lumen Reflections の有効/無効 |
| `r.Lumen.Reflections.DownsampleFactor` | 1 | トレース解像度ダウンサンプル係数 |
| `r.Lumen.Reflections.DownsampleCheckerboard` | 0 | チェッカーボードダウンサンプルの使用 |
| `r.Lumen.Reflections.MaxRoughnessToTrace` | -1.0 | トレースする最大ラフネス（-1=PostProcess設定を使用）|
| `r.Lumen.Reflections.MaxRoughnessToTraceClamp` | 1.0 | スケーラビリティ側の最大ラフネス上限 |
| `r.Lumen.Reflections.MaxRoughnessToTraceForFoliage` | 0.2 | フォリッジ用最大ラフネス |
| `r.Lumen.Reflections.RoughnessFadeLength` | 0.1 | ラフネスフェード幅 |
| `r.Lumen.Reflections.GGXSamplingBias` | 0.1 | GGX サンプリングバイアス |
| `r.Lumen.Reflections.Temporal` | 1 | テンポラルフィルタの有効/無効 |
| `r.Lumen.Reflections.Temporal.MaxFramesAccumulated` | 12.0 | テンポラル最大蓄積フレーム数 |
| `r.Lumen.Reflections.TraceMeshSDFs` | 1 | Mesh SDF トレースの有効/無効 |
| `r.Lumen.Reflections.RadianceCache` | 0 | Radiance Cache の使用（粗面の反射レイを短縮）|
| `r.Lumen.Reflections.RadianceCache.MinRoughness` | 0.2 | Radiance Cache を使い始めるラフネス閾値 |
| `r.Lumen.Reflections.RadianceCache.MaxRoughness` | 0.35 | Radiance Cache に完全切り替えるラフネス閾値 |
| `r.Lumen.Reflections.RadianceCache.MinTraceDistance` | 1000.0 | Radiance Cache 切り替え前の最小トレース距離 |
| `r.Lumen.Reflections.RadianceCache.MaxTraceDistance` | 5000.0 | Radiance Cache 切り替え前の最大トレース距離 |
