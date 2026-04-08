# リファレンス：LumenScreenProbeGather.h / LumenScreenProbeGather.cpp

- グループ: e - Diffuse GI
- 上位: [[e_lumen_diffuse_gi]]
- 関連: [[ref_lumen_diffuse_indirect]] | [[ref_lumen_screen_probe_tracing]] | [[ref_lumen_radiance_cache]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenScreenProbeGather.h/cpp`

---

## 概要

Lumen の拡散 GI を担う **Screen Probe Gather** システムの中核ヘッダ・実装ファイル。  
スクリーン空間タイルに放射照度プローブを配置し、SDF/HW RT でトレースしてフィルタ・統合する。  
プローブは**均一（Uniform）**と**適応（Adaptive）**の 2 種類を持つ。

---

## EScreenProbeIrradianceFormat

プローブが格納する照度形式。

```cpp
enum class EScreenProbeIrradianceFormat : uint8 {
    SH3,        // L1 球面調和関数（3 係数）
    Octahedral, // 八面体マッピング（方向テクスチャ）
    MAX
};
```

---

## FScreenProbeParameters

Screen Probe のトレース・フィルタ全体で共有する主要パラメータ構造体。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FScreenProbeParameters, )
    // プローブ解像度
    SHADER_PARAMETER(uint32, ScreenProbeTracingOctahedronResolution) // トレース八面体解像度
    SHADER_PARAMETER(uint32, ScreenProbeGatherOctahedronResolution)  // Gather 八面体解像度
    SHADER_PARAMETER(uint32, ScreenProbeGatherOctahedronResolutionWithBorder)
    SHADER_PARAMETER(uint32, ScreenProbeDownsampleFactor) // プローブ間のタイルサイズ（px）
    SHADER_PARAMETER(FIntPoint, ScreenProbeViewSize)      // ビューのプローブ数
    SHADER_PARAMETER(FIntPoint, ScreenProbeAtlasViewSize) // アトラスのビュー領域
    SHADER_PARAMETER(FIntPoint, ScreenProbeAtlasBufferSize)

    // ガーサー設定
    SHADER_PARAMETER(float, ScreenProbeGatherMaxMip)
    SHADER_PARAMETER(float, RelativeSpeedDifferenceToConsiderLightingMoving)
    SHADER_PARAMETER(float, ScreenTraceNoFallbackThicknessScale)
    SHADER_PARAMETER(float, ExtraAOMaxDistanceWorldSpace)
    SHADER_PARAMETER(float, ExtraAOExponent)
    SHADER_PARAMETER(float, ScreenProbeInterpolationDepthWeight)
    SHADER_PARAMETER(float, ScreenProbeInterpolationDepthWeightForFoliage)

    // UV スケール（Radiance Cache サンプリング用）
    SHADER_PARAMETER(FVector2f, SampleRadianceProbeUVMul)
    SHADER_PARAMETER(FVector2f, SampleRadianceProbeUVAdd)
    SHADER_PARAMETER(FVector2f, SampleRadianceAtlasUVMul)

    // プローブ数
    SHADER_PARAMETER(uint32, AdaptiveScreenTileSampleResolution)
    SHADER_PARAMETER(uint32, NumUniformScreenProbes)
    SHADER_PARAMETER(uint32, MaxNumAdaptiveProbes)
    SHADER_PARAMETER(int32,  FixedJitterIndex)
    SHADER_PARAMETER(uint32, ScreenProbeRayDirectionFrameIndex)
    SHADER_PARAMETER(uint32, bSupportsHairScreenTraces)
    SHADER_PARAMETER(FVector3f, TargetFormatQuantizationError)

    // 適応プローブバッファ
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, NumAdaptiveScreenProbes)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, AdaptiveScreenProbeData)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenTileAdaptiveProbeHeader)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenTileAdaptiveProbeIndices)

    // トレース結果
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, TraceRadiance)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, TraceHit)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float3>, RWTraceRadiance)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<uint>,   RWTraceHit)

    // プローブ座標
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenProbeSceneDepth)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenProbeWorldNormal)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenProbeWorldSpeed)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenProbeTranslatedWorldPosition)

    // 重要度サンプリング
    SHADER_PARAMETER_STRUCT_INCLUDE(FScreenProbeImportanceSamplingParameters, ImportanceSampling)
    SHADER_PARAMETER_STRUCT_REF(FBlueNoise, BlueNoise)
    RDG_BUFFER_ACCESS(ProbeIndirectArgs, ERHIAccess::IndirectArgs)
END_SHADER_PARAMETER_STRUCT()
```

---

## FScreenProbeGatherParameters

Gather（フィルタ後の照度テクスチャ）への参照パラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FScreenProbeGatherParameters, )
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenProbeRadiance)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenProbeRadianceWithBorder)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float3>, ScreenProbeRadianceSHAmbient)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float4>, ScreenProbeRadianceSHDirectional)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float3>, ScreenProbeIrradianceWithBorder)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float3>, ScreenProbeExtraAOWithBorder)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float>,  ScreenProbeMoving)
END_SHADER_PARAMETER_STRUCT()
```

---

## EScreenProbeIndirectArgs / EScreenProbeIntegrateTileClassification

```cpp
// Indirect Dispatch の引数オフセット（SetupAdaptiveProbeIndirectArgsCS と対応）
enum class EScreenProbeIndirectArgs {
    GroupPerProbe, ThreadPerProbe, TraceCompaction,
    ThreadPerTrace, ThreadPerGather, ThreadPerGatherWithBorder, Max
};

// タイル分類（VGPR 使用量で最適な Dispatch を選択）
enum class EScreenProbeIntegrateTileClassification {
    SimpleDiffuse,              // 単純拡散
    SupportImportanceSampleBRDF, // BRDF 重要度サンプリング
    SupportAll,                  // 全機能サポート
    Num
};
```

---

## LumenScreenProbeGather 名前空間

### 判定・設定関数

| 関数 | 説明 |
|-----|------|
| `GetTracingOctahedronResolution(View)` | トレース八面体解像度を返す |
| `IsProbeTracingResolutionSupportedForImportanceSampling(TracingResolution)` | 重要度サンプリングが使える解像度か |
| `UseImportanceSampling(View)` | BRDF 重要度サンプリングを使うか |
| `UseProbeSpatialFilter()` | プローブ空間フィルタを使うか |
| `UseProbeTemporalFilter()` | プローブテンポラルフィルタを使うか |
| `UseRadianceCache()` | Radiance Cache を使うか |
| `UseRadianceCacheSkyVisibility()` | Sky Visibility を Radiance Cache で計算するか |
| `GetScreenProbeIrradianceFormat(ShowFlags)` | 照度形式（SH3 / Octahedral）を返す |
| `UseScreenProbeExtraAO()` | ExtraAO（追加 AO）を使うか |
| `GetStateFrameIndex(ViewState)` | フレームインデックス（テンポラル用）を返す |
| `GetRequestedIntegrateDownsampleFactor()` | 統合ダウンサンプル係数を返す |
| `SetupTileClassifyParameters(View, OutParameters)` | タイル分類パラメータをセットアップ |
| `IsUsingDownsampledDepthAndNormal(View)` | 深度・法線をダウンサンプルして使うか |

### 定数

```cpp
constexpr uint32 IrradianceProbeRes           = 6;  // 照度プローブの解像度
constexpr uint32 IrradianceProbeWithBorderRes = 8;  // ボーダー付き照度プローブ解像度（6+2）
```

---

## 主要外部関数

```cpp
// トレース結果をコンパクト化（距離でカリングしてカウンタバッファに積む）
FCompactedTraceParameters LumenScreenProbeGather::CompactTraces(
    FRDGBuilder&, const FViewInfo&, const FScreenProbeParameters&,
    bool bCullByDistanceFromCamera, float CompactionTracingEndDistanceFromCamera,
    float CompactionMaxTraceDistance, bool bCompactForSkyApply, ERDGPassFlags);

// GBuffer から BRDF 確率密度関数（PDF）を生成
void GenerateBRDF_PDF(
    FRDGBuilder&, const FViewInfo&, const FSceneTextures&,
    FRDGTextureRef& BRDFProbabilityDensityFunction,
    FRDGBufferSRVRef& BRDFProbabilityDensityFunctionSH,
    FScreenProbeParameters&, ERDGPassFlags);

// 重要度サンプリングレイを生成
void GenerateImportanceSamplingRays(
    FRDGBuilder&, const FViewInfo&, const FSceneTextures&,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters&,
    FRDGTextureRef BRDFProbabilityDensityFunction,
    FRDGBufferSRVRef BRDFProbabilityDensityFunctionSH,
    FScreenProbeParameters&, ERDGPassFlags);

// Screen Probe のトレースを実行（SW RT: SDF + HZB Screen Trace）
void TraceScreenProbes(
    FRDGBuilder&, const FScene*, const FViewInfo&,
    const FLumenSceneFrameTemporaries&, bool bTraceMeshObjects,
    const FSceneTextures&, FRDGTextureRef LightingChannelsTexture,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters&,
    FScreenProbeParameters&, FLumenMeshSDFGridParameters&, ERDGPassFlags);

// Screen Probe のトレースを実行（HW RT）
void RenderHardwareRayTracingScreenProbe(
    FRDGBuilder&, const FScene*, const FSceneTextureParameters&,
    FScreenProbeParameters&, const FViewInfo&,
    const FLumenCardTracingParameters&, FLumenIndirectTracingParameters&,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters&, ERDGPassFlags);
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.ScreenProbeGather` | 1 | Screen Probe Gather の有効/無効 |
| `r.Lumen.ScreenProbeGather.WaveOps` | 1 | Wave Ops（SIMD グループ操作）の使用 |
| `r.Lumen.ScreenProbeGather.DownsampleFactor` | 16 | プローブを置くタイルサイズ（px）|
| `r.Lumen.ScreenProbeGather.TracingOctahedronResolution` | 8 | トレース方向数（8×8=64 レイ/プローブ）|
| `r.Lumen.ScreenProbeGather.GatherOctahedronResolutionScale` | 1.0 | Gather 解像度スケール |
| `r.Lumen.ScreenProbeGather.NumAdaptiveProbes` | 8 | 均一プローブ 1 枚あたりの最大適応プローブ数 |
| `r.Lumen.ScreenProbeGather.AdaptiveProbeAllocationFraction` | 0.5 | 適応プローブに割り当てる割合 |
| `r.Lumen.ScreenProbeGather.Temporal` | 1 | テンポラルフィルタの有効/無効 |
| `r.Lumen.ScreenProbeGather.DiffuseIntegralMethod` | 0 | 統合方式（0=事前積分, 1=BRDF IS, 2=数値積分参照）|
| `r.Lumen.ScreenProbeGather.InterpolationDepthWeight` | 1.0 | 深度テストの重み（大きいほどシャープだが不安定）|
| `r.Lumen.ScreenProbeGather.InterpolationDepthWeightForFoliage` | 0.25 | フォリッジ用の深度重み（リーク抑制を緩める）|
| `r.Lumen.ScreenProbeGather.TwoSidedFoliageBackfaceDiffuse` | 1 | Two Sided Foliage の裏面照明の計算 |
| `r.Lumen.ScreenProbeGather.MaterialAO` | 1 | マテリアル AO / Bent Normal を GI に適用するか |
| `r.Lumen.ScreenProbeGather.ReferenceMode` | 0 | 参照モード（1024 レイ均一トレース、デバッグ用）|
