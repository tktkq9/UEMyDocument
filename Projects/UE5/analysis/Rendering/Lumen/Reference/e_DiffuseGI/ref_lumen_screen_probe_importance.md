# リファレンス：LumenScreenProbeImportanceSampling.cpp

- グループ: e - Diffuse GI
- 上位: [[e_lumen_diffuse_gi]]
- 関連: [[ref_lumen_screen_probe_gather]] | [[ref_lumen_screen_probe_tracing]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenScreenProbeImportanceSampling.cpp`

---

## 概要

Screen Probe の **BRDF 重要度サンプリング**（Importance Sampling）実装ファイル。  
GBuffer の BRDF（ラフネス・法線）から確率密度関数（PDF）を計算し、  
レイ方向を BRDF の形状に沿って集中させることでトレース効率を大幅に向上させる。  
`UseImportanceSampling()` が true の場合（高いラフネスを持つ鏡面反射があるとき）に有効化される。

---

## FScreenProbeImportanceSamplingParameters

重要度サンプリングに必要なシェーダーパラメータ。  
`FScreenProbeParameters.ImportanceSampling` としてインクルードされる。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FScreenProbeImportanceSamplingParameters, )
    // 最大重要度サンプリング八面体解像度（TracingOctahedronResolution × 何倍まで許容するか）
    SHADER_PARAMETER(uint32, MaxImportanceSamplingOctahedronResolution)

    // BRDF 確率密度関数の八面体解像度
    SHADER_PARAMETER(uint32, ScreenProbeBRDFOctahedronResolution)

    // 構造化重要度サンプリングレイ情報（トレース方向テクスチャ）
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<uint>, StructuredImportanceSampledRayInfosForTracing)
END_SHADER_PARAMETER_STRUCT()
```

---

## 主要外部関数

### GenerateBRDF_PDF

GBuffer（ラフネス・法線・メタリック）から BRDF の確率密度関数を生成する。

```cpp
void GenerateBRDF_PDF(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    FRDGTextureRef& BRDFProbabilityDensityFunction,      // 出力: PDF テクスチャ
    FRDGBufferSRVRef& BRDFProbabilityDensityFunctionSH, // 出力: PDF の SH バッファ
    FScreenProbeParameters& ScreenProbeParameters,
    ERDGPassFlags ComputePassFlags);
```

### GenerateImportanceSamplingRays

BRDF PDF と Radiance Cache を組み合わせてレイ方向を生成する。

```cpp
void GenerateImportanceSamplingRays(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters& RadianceCacheParameters,
    FRDGTextureRef BRDFProbabilityDensityFunction,
    FRDGBufferSRVRef BRDFProbabilityDensityFunctionSH,
    FScreenProbeParameters& ScreenProbeParameters,
    ERDGPassFlags ComputePassFlags);
// → StructuredImportanceSampledRayInfosForTracing に書き込み
// → トレーシングシェーダーがこのテクスチャを読んでレイ方向を決定
```

---

## LumenScreenProbeGather 名前空間（IS 関連）

```cpp
namespace LumenScreenProbeGather {
    // BRDF 重要度サンプリングが使える TracingOctahedronResolution か（8, 16, 32 等）
    int32 IsProbeTracingResolutionSupportedForImportanceSampling(int32 TracingResolution);

    // 重要度サンプリングを使うか
    // 条件: CVar 有効 && 解像度が対応 && Radiance Cache 利用可能
    bool UseImportanceSampling(const FViewInfo& View);
}
```

---

## EScreenProbeIntegrateTileClassification との連携

```
タイル分類 EScreenProbeIntegrateTileClassification:
  ├─ SimpleDiffuse                → 均一拡散のみ（IS 不要）
  ├─ SupportImportanceSampleBRDF → BRDF IS が必要（高ラフネス）
  └─ SupportAll                  → 全機能（低ラフネス鏡面含む）

IS は SupportImportanceSampleBRDF / SupportAll のタイルのみで実行
  → VGPR が多くなるが精度が向上
  → SimpleDiffuse タイルは IS をスキップしてコストを下げる
```

---

## BRDF PDF 重要度サンプリングの仕組み

```
GenerateBRDF_PDF():
  GBuffer.Roughness, GBuffer.Normal → GGX BRDF の PDF を計算
  → PDF をオクタヒドラルマップ（ScreenProbeBRDFOctahedronResolution）に投影
  → ハイライト（低ラフネス方向）ほど高い確率値が割り当てられる

GenerateImportanceSamplingRays():
  PDF + Radiance Cache SH（方向ヒントとして使用）
  → PDF に従って TracingOctahedronResolution の方向をリマッピング
  → StructuredImportanceSampledRayInfosForTracing に格納

TraceScreenProbes():
  IS が有効なプローブでは均一八面体方向の代わりに
  StructuredImportanceSampledRayInfosForTracing のレイ方向を使用
  → ハイライト方向に多くのレイを集中 → 少ないレイで収束
```
