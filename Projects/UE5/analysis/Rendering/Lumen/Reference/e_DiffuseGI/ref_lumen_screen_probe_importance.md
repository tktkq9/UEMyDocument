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
    SHADER_PARAMETER(uint32, MaxImportanceSamplingOctahedronResolution)
    SHADER_PARAMETER(uint32, ScreenProbeBRDFOctahedronResolution)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<uint>, StructuredImportanceSampledRayInfosForTracing)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `MaxImportanceSamplingOctahedronResolution` | `uint32` | 重要度サンプリングの最大八面体解像度。TracingOctahedronResolution の倍数で制限される |
| `ScreenProbeBRDFOctahedronResolution` | `uint32` | BRDF PDF テクスチャの八面体解像度（GGX の形状に応じて決まる）|
| `StructuredImportanceSampledRayInfosForTracing` | `Texture2D<uint>` | 構造化 IS レイ情報テクスチャ。各テクセルに { リマップ後の方向インデックス, 重み } を格納 |

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — `FScreenProbeParameters.ImportanceSampling` にインクルードされ全トレースシェーダーで参照
- [[ref_lumen_screen_probe_tracing]] — `TraceScreenProbes()` がこのパラメータを読んで IS レイ方向を決定
- [[ref_lumen_screen_probe_hwrt]] — HW RT トレースでも同様に参照

---

## GenerateBRDF_PDF

GBuffer（ラフネス・法線・メタリック）から BRDF の確率密度関数を生成する。

```cpp
void GenerateBRDF_PDF(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    FRDGTextureRef& BRDFProbabilityDensityFunction,
    FRDGBufferSRVRef& BRDFProbabilityDensityFunctionSH,
    FScreenProbeParameters& ScreenProbeParameters,
    ERDGPassFlags ComputePassFlags);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `View` | `const FViewInfo&` | カメラビュー |
| `SceneTextures` | `const FSceneTextures&` | GBuffer（ラフネス・法線・メタリックの参照元）|
| `BRDFProbabilityDensityFunction` | `FRDGTextureRef&` | 出力: BRDF PDF テクスチャ（各プローブの八面体マップ）|
| `BRDFProbabilityDensityFunctionSH` | `FRDGBufferSRVRef&` | 出力: PDF の SH バッファ（Radiance Cache 方向ヒントとの組み合わせ用）|
| `ScreenProbeParameters` | `FScreenProbeParameters&` | `ImportanceSampling.ScreenProbeBRDFOctahedronResolution` などを更新 |
| `ComputePassFlags` | `ERDGPassFlags` | AsyncCompute / Compute の選択フラグ |

### 内部処理フロー

1. **GBuffer サンプリング**
   ```cpp
   // 各プローブ位置の GBuffer.Roughness / GBuffer.Normal / GBuffer.Metallic を読み取り
   ```

2. **GGX BRDF PDF の計算**
   ```cpp
   // GGX NDF（Normal Distribution Function）を八面体マップに投影
   // → ハイライト方向（鏡面反射ローブ）に高い確率値を割り当て
   // → BRDFProbabilityDensityFunction テクスチャ（Texture2D）に書き込み
   ```

3. **PDF → SH 変換**
   ```cpp
   // PDF を SH（球面調和）に射影 → BRDFProbabilityDensityFunctionSH バッファに書き込み
   // → Radiance Cache の方向分布と組み合わせて IS レイ方向を生成するために使用
   ```

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — `UseImportanceSampling()` が true の場合に `GenerateImportanceSamplingRays()` の前に呼ばれる
- [[ref_lumen_radiance_cache]] — `BRDFProbabilityDensityFunctionSH` が `FUpdateInputs::BRDFProbabilityDensityFunctionSH` として渡される

---

## GenerateImportanceSamplingRays

BRDF PDF と Radiance Cache SH を組み合わせてレイ方向を生成する。

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
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `RadianceCacheParameters` | `FRadianceCacheInterpolationParameters` | Radiance Cache の方向分布（照明の来る方向ヒント）|
| `BRDFProbabilityDensityFunction` | `FRDGTextureRef` | GenerateBRDF_PDF() で生成した PDF テクスチャ |
| `BRDFProbabilityDensityFunctionSH` | `FRDGBufferSRVRef` | GenerateBRDF_PDF() で生成した PDF SH バッファ |
| `ScreenProbeParameters` | `FScreenProbeParameters&` | `ImportanceSampling.StructuredImportanceSampledRayInfosForTracing` を更新 |

### 内部処理フロー

1. **BRDF PDF × Radiance Cache SH の組み合わせ**
   ```cpp
   // PDF（BRDF の形状）と Radiance Cache SH（照明の方向分布）を乗算
   // → 光が来る方向 かつ BRDF が反応する方向 に高い重みを付ける
   CombinedPDF = BRDF_PDF * RadianceCacheSH;
   ```

2. **IS レイ方向のリマッピング**
   ```cpp
   // 均一八面体の方向インデックスを CombinedPDF に従ってリマッピング
   // → 高確率方向に多くのレイを集中させる
   // → TracingOctahedronResolution と ScreenProbeBRDFOctahedronResolution の両方を考慮
   ```

3. **StructuredImportanceSampledRayInfosForTracing に格納**
   ```cpp
   // Texture2D<uint> に { リマップ後の方向インデックス, 重みの逆数（サンプリング補正用）} を書き込み
   // → トレースシェーダーがこれを参照して均一八面体の代わりに IS 方向を使用
   ```

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — `GenerateBRDF_PDF()` 直後に呼ばれる
- [[ref_lumen_screen_probe_tracing]] — `StructuredImportanceSampledRayInfosForTracing` をトレースシェーダーが参照

---

## LumenScreenProbeGather 名前空間（IS 関連）

```cpp
namespace LumenScreenProbeGather {
    int32 IsProbeTracingResolutionSupportedForImportanceSampling(int32 TracingResolution);
    bool UseImportanceSampling(const FViewInfo& View);
}
```

### 関数説明

| 関数 | 戻り値 | 説明 |
|------|--------|------|
| `IsProbeTracingResolutionSupportedForImportanceSampling(Res)` | `int32` | 非ゼロ = 対応。8, 16, 32 等の 2 の冪のみ対応 |
| `UseImportanceSampling(View)` | `bool` | IS を使うか。条件: CVar 有効 + 解像度対応 + Radiance Cache 利用可能 |

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — `RenderLumenScreenProbeGather()` で IS パスを追加するか判定
- [[ref_lumen_screen_probe_tracing]] — トレース前に IS レイを使うか切り替え

---

## BRDF PDF 重要度サンプリングの全体フロー

```
GenerateBRDF_PDF():
  GBuffer.Roughness, GBuffer.Normal → GGX BRDF の PDF を計算
  → PDF をオクタヒドラルマップに投影
  → ハイライト（低ラフネス方向）ほど高い確率値が割り当てられる

GenerateImportanceSamplingRays():
  PDF + Radiance Cache SH（方向ヒントとして使用）
  → PDF に従って TracingOctahedronResolution の方向をリマッピング
  → StructuredImportanceSampledRayInfosForTracing に格納

TraceScreenProbes() / RenderHardwareRayTracingScreenProbe():
  IS が有効なプローブでは均一八面体方向の代わりに
  StructuredImportanceSampledRayInfosForTracing のレイ方向を使用
  → ハイライト方向に多くのレイを集中 → 少ないレイで収束
  → EScreenProbeIntegrateTileClassification で IS 不要タイルをスキップ
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
