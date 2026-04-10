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

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — `LumenScreenProbeGather::GetScreenProbeIrradianceFormat()` で ShowFlags から決定
- [[ref_lumen_screen_probe_filtering]] — フィルタパスの SH 射影 vs 八面体マップ出力の分岐に使用

---

## FScreenProbeParameters

Screen Probe のトレース・フィルタ全体で共有する主要パラメータ構造体。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FScreenProbeParameters, )
    SHADER_PARAMETER(uint32, ScreenProbeTracingOctahedronResolution)
    SHADER_PARAMETER(uint32, ScreenProbeGatherOctahedronResolution)
    SHADER_PARAMETER(uint32, ScreenProbeGatherOctahedronResolutionWithBorder)
    SHADER_PARAMETER(uint32, ScreenProbeDownsampleFactor)
    SHADER_PARAMETER(FIntPoint, ScreenProbeViewSize)
    SHADER_PARAMETER(FIntPoint, ScreenProbeAtlasViewSize)
    SHADER_PARAMETER(FIntPoint, ScreenProbeAtlasBufferSize)
    SHADER_PARAMETER(float, ScreenProbeGatherMaxMip)
    SHADER_PARAMETER(float, RelativeSpeedDifferenceToConsiderLightingMoving)
    SHADER_PARAMETER(float, ScreenTraceNoFallbackThicknessScale)
    SHADER_PARAMETER(float, ExtraAOMaxDistanceWorldSpace)
    SHADER_PARAMETER(float, ExtraAOExponent)
    SHADER_PARAMETER(float, ScreenProbeInterpolationDepthWeight)
    SHADER_PARAMETER(float, ScreenProbeInterpolationDepthWeightForFoliage)
    SHADER_PARAMETER(FVector2f, SampleRadianceProbeUVMul)
    SHADER_PARAMETER(FVector2f, SampleRadianceProbeUVAdd)
    SHADER_PARAMETER(FVector2f, SampleRadianceAtlasUVMul)
    SHADER_PARAMETER(uint32, AdaptiveScreenTileSampleResolution)
    SHADER_PARAMETER(uint32, NumUniformScreenProbes)
    SHADER_PARAMETER(uint32, MaxNumAdaptiveProbes)
    SHADER_PARAMETER(int32,  FixedJitterIndex)
    SHADER_PARAMETER(uint32, ScreenProbeRayDirectionFrameIndex)
    SHADER_PARAMETER(uint32, bSupportsHairScreenTraces)
    SHADER_PARAMETER(FVector3f, TargetFormatQuantizationError)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, NumAdaptiveScreenProbes)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, AdaptiveScreenProbeData)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenTileAdaptiveProbeHeader)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenTileAdaptiveProbeIndices)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, TraceRadiance)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, TraceHit)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float3>, RWTraceRadiance)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<uint>,   RWTraceHit)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenProbeSceneDepth)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenProbeWorldNormal)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenProbeWorldSpeed)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ScreenProbeTranslatedWorldPosition)
    SHADER_PARAMETER_STRUCT_INCLUDE(FScreenProbeImportanceSamplingParameters, ImportanceSampling)
    SHADER_PARAMETER_STRUCT_REF(FBlueNoise, BlueNoise)
    RDG_BUFFER_ACCESS(ProbeIndirectArgs, ERHIAccess::IndirectArgs)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `ScreenProbeTracingOctahedronResolution` | `uint32` | トレース八面体解像度（例: 8 → 8×8=64 レイ/プローブ）|
| `ScreenProbeGatherOctahedronResolution` | `uint32` | フィルタ後 Gather 八面体解像度（TracingResolution から算出）|
| `ScreenProbeGatherOctahedronResolutionWithBorder` | `uint32` | ボーダー込みの Gather 解像度（GatherRes + 2）|
| `ScreenProbeDownsampleFactor` | `uint32` | プローブを置くタイルサイズ（px）。`r.Lumen.ScreenProbeGather.DownsampleFactor`|
| `ScreenProbeViewSize` | `FIntPoint` | ビューのプローブ数（X = View.Width / DownsampleFactor）|
| `ScreenProbeAtlasViewSize` | `FIntPoint` | アトラス内のビュー領域サイズ |
| `ScreenProbeAtlasBufferSize` | `FIntPoint` | アトラスバッファ全体サイズ（均一 + 適応プローブ分）|
| `ScreenProbeGatherMaxMip` | `float` | Gather テクスチャの最大ミップ |
| `RelativeSpeedDifferenceToConsiderLightingMoving` | `float` | この速度差以上なら「照明が動いている」とみなすしきい値 |
| `ScreenTraceNoFallbackThicknessScale` | `float` | HZB スクリーントレースのハーフスペース厚みスケール |
| `ExtraAOMaxDistanceWorldSpace` | `float` | Extra AO の最大距離（ワールド空間 cm）|
| `ExtraAOExponent` | `float` | Extra AO の指数（1.0=線形, 2.0=二乗）|
| `ScreenProbeInterpolationDepthWeight` | `float` | 深度テストの重み（大きいほどシャープ）|
| `ScreenProbeInterpolationDepthWeightForFoliage` | `float` | フォリッジ用の深度重み（リーク緩和）|
| `SampleRadianceProbeUVMul` / `Add` | `FVector2f` | Radiance Cache サンプリング用 UV 変換係数 |
| `SampleRadianceAtlasUVMul` | `FVector2f` | Radiance アトラス UV 変換係数 |
| `AdaptiveScreenTileSampleResolution` | `uint32` | 適応プローブのタイル内サンプル解像度 |
| `NumUniformScreenProbes` | `uint32` | 均一プローブの総数（View.X/Factor × View.Y/Factor）|
| `MaxNumAdaptiveProbes` | `uint32` | 最大適応プローブ数（NumUniform × NumAdaptiveProbes CVar）|
| `FixedJitterIndex` | `int32` | 固定ジッターインデックス（-1=フレーム毎に変化）|
| `ScreenProbeRayDirectionFrameIndex` | `uint32` | レイ方向のフレームインデックス（テンポラル回転）|
| `bSupportsHairScreenTraces` | `uint32` | Hair Strands の HZB スクリーントレースを使うか |
| `TargetFormatQuantizationError` | `FVector3f` | 出力フォーマットの量子化誤差（精度調整用）|
| `NumAdaptiveScreenProbes` | `Buffer<uint>` | 適応プローブの実際の数（Compute パスで更新）|
| `AdaptiveScreenProbeData` | `Buffer<uint>` | 適応プローブのプローブインデックス・タイル情報 |
| `ScreenTileAdaptiveProbeHeader` | `Texture2D` | 各タイルの適応プローブ先頭インデックス |
| `ScreenTileAdaptiveProbeIndices` | `Texture2D` | タイルごとの適応プローブインデックスリスト |
| `TraceRadiance` | `Texture2D` | トレース結果: Radiance（読み取り用）|
| `TraceHit` | `Texture2D` | トレース結果: ヒット距離・法線（読み取り用）|
| `RWTraceRadiance` | `RWTexture2D<float3>` | トレース結果 UAV（書き込み用）|
| `RWTraceHit` | `RWTexture2D<uint>` | トレースヒット UAV（書き込み用）|
| `ScreenProbeSceneDepth` | `Texture2D` | プローブ位置の Scene Depth |
| `ScreenProbeWorldNormal` | `Texture2D` | プローブ位置のワールド法線 |
| `ScreenProbeWorldSpeed` | `Texture2D` | プローブ位置のワールド速度（Motion Vector）|
| `ScreenProbeTranslatedWorldPosition` | `Texture2D` | プローブのワールド位置（TranslatedWorld 空間）|
| `ImportanceSampling` | `FScreenProbeImportanceSamplingParameters` | BRDF 重要度サンプリングパラメータ（[[ref_lumen_screen_probe_importance]] 参照）|
| `BlueNoise` | `FBlueNoise` | ブルーノイズテクスチャ（ジッター用）|
| `ProbeIndirectArgs` | `IndirectArgs` | プローブ数ベースの Indirect Dispatch Args バッファ |

### 使用箇所

- [[ref_lumen_screen_probe_tracing]] — `TraceScreenProbes()` の入出力引数
- [[ref_lumen_screen_probe_hwrt]] — `RenderHardwareRayTracingScreenProbe()` の入出力引数
- [[ref_lumen_screen_probe_filtering]] — フィルタ・統合パスで TraceRadiance / TraceHit を読み取り
- [[ref_lumen_screen_probe_importance]] — `GenerateBRDF_PDF()` / `GenerateImportanceSamplingRays()` が `ImportanceSampling` を更新

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

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `ScreenProbeRadiance` | `Texture2D` | フィルタ済みプローブ Radiance（統合パスのソース）|
| `ScreenProbeRadianceWithBorder` | `Texture2D` | ボーダー付きプローブ Radiance（隣接プローブ補間用）|
| `ScreenProbeRadianceSHAmbient` | `Texture2D<float3>` | SH 環境成分テクスチャ（IrradianceFormat=SH3 の場合）|
| `ScreenProbeRadianceSHDirectional` | `Texture2D<float4>` | SH 指向成分テクスチャ（IrradianceFormat=SH3 の場合）|
| `ScreenProbeIrradianceWithBorder` | `Texture2D<float3>` | ボーダー付き照度テクスチャ（IrradianceFormat=Octahedral の場合）|
| `ScreenProbeExtraAOWithBorder` | `Texture2D<float3>` | Extra AO テクスチャ（UseScreenProbeExtraAO()=true の場合）|
| `ScreenProbeMoving` | `Texture2D<float>` | プローブ移動フラグ（0=静的, 1=動的。テンポラルブレンド率制御）|

### 使用箇所

- [[ref_lumen_screen_probe_filtering]] — フィルタパスが結果をこの構造体に書き込む
- 統合パス（LumenScreenProbeIntegrate.cpp）— ScreenProbeGatherParameters から照度を補間して各ピクセルに適用

---

## EScreenProbeIndirectArgs / EScreenProbeIntegrateTileClassification

```cpp
// Indirect Dispatch の引数オフセット（SetupAdaptiveProbeIndirectArgsCS と対応）
enum class EScreenProbeIndirectArgs {
    GroupPerProbe,               // 1 プローブ = 1 ThreadGroup
    ThreadPerProbe,              // 1 プローブ = 1 スレッド
    TraceCompaction,             // Compact パス用
    ThreadPerTrace,              // 1 トレース = 1 スレッド
    ThreadPerGather,             // Gather 解像度のスレッド
    ThreadPerGatherWithBorder,   // ボーダー込み Gather 解像度
    Max
};

// タイル分類（VGPR 使用量で最適な Dispatch を選択）
enum class EScreenProbeIntegrateTileClassification {
    SimpleDiffuse,               // 単純拡散（重要度サンプリング不要）
    SupportImportanceSampleBRDF, // BRDF 重要度サンプリングが必要（高ラフネス鏡面）
    SupportAll,                  // 全機能サポート（低ラフネス鏡面含む）
    Num
};
```

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — `SetupAdaptiveProbeIndirectArgsCS` が `EScreenProbeIndirectArgs` に対応したバッファを構築
- [[ref_lumen_screen_probe_filtering]] — 統合パスが `EScreenProbeIntegrateTileClassification` に応じてシェーダーを切り替え

---

## LumenScreenProbeGather 名前空間

### 判定・設定関数

| 関数 | 戻り値 | 説明 |
|------|--------|------|
| `GetTracingOctahedronResolution(View)` | `uint32` | `r.Lumen.ScreenProbeGather.TracingOctahedronResolution` CVar 値 |
| `IsProbeTracingResolutionSupportedForImportanceSampling(Res)` | `bool` | 解像度が IS に対応しているか（8, 16, 32 等の 2 の冪）|
| `UseImportanceSampling(View)` | `bool` | BRDF 重要度サンプリングを使うか（CVar + 解像度 + Radiance Cache 全て有効時）|
| `UseProbeSpatialFilter()` | `bool` | `r.Lumen.ScreenProbeGather.SpatialFilterProbes` が有効か |
| `UseProbeTemporalFilter()` | `bool` | `r.Lumen.ScreenProbeGather.Temporal` が有効か |
| `UseRadianceCache()` | `bool` | Radiance Cache を Screen Probe に使うか（`r.Lumen.ScreenProbeGather.RadianceCache`）|
| `UseRadianceCacheSkyVisibility()` | `bool` | Sky Visibility を Radiance Cache で計算するか |
| `GetScreenProbeIrradianceFormat(ShowFlags)` | `EScreenProbeIrradianceFormat` | 照度形式（SH3 / Octahedral）|
| `UseScreenProbeExtraAO()` | `bool` | Extra AO（追加 AO）を計算するか |
| `GetStateFrameIndex(ViewState)` | `uint32` | テンポラル用フレームインデックス |
| `GetRequestedIntegrateDownsampleFactor()` | `uint32` | 統合パスのダウンサンプル係数 |
| `SetupTileClassifyParameters(View, OutParams)` | `void` | タイル分類パラメータを構築 |
| `IsUsingDownsampledDepthAndNormal(View)` | `bool` | 深度・法線をダウンサンプルして使うか |

### 使用箇所（名前空間全体）

- [[ref_lumen_diffuse_indirect]] — `RenderLumenScreenProbeGather()` がこれらの関数で動作を決定
- [[ref_lumen_screen_probe_tracing]] — トレース実行前に `UseImportanceSampling()` / `UseRadianceCache()` を確認
- [[ref_lumen_screen_probe_filtering]] — フィルタパスが `UseProbeSpatialFilter()` / `UseProbeTemporalFilter()` を確認

### 定数

```cpp
constexpr uint32 IrradianceProbeRes           = 6;  // 照度プローブの解像度
constexpr uint32 IrradianceProbeWithBorderRes = 8;  // ボーダー付き照度プローブ解像度（6+2）
```

**使用箇所**: [[ref_lumen_screen_probe_filtering]] — SH 射影後の照度アトラスのレイアウト計算に使用

---

## 主要外部関数

### CompactTraces

```cpp
FCompactedTraceParameters LumenScreenProbeGather::CompactTraces(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FScreenProbeParameters& ScreenProbeParameters,
    bool bCullByDistanceFromCamera,
    float CompactionTracingEndDistanceFromCamera,
    float CompactionMaxTraceDistance,
    bool bCompactForSkyApply,
    ERDGPassFlags ComputePassFlags);
```

| 引数 | 型 | 説明 |
|------|-----|------|
| `bCullByDistanceFromCamera` | `bool` | カメラ距離でカリングするか |
| `CompactionTracingEndDistanceFromCamera` | `float` | カメラからのトレース最大距離（cm）|
| `CompactionMaxTraceDistance` | `float` | レイの最大トレース距離（cm）|
| `bCompactForSkyApply` | `bool` | スカイ適用用（別パス）のコンパクト化か |

**戻り値**: `FCompactedTraceParameters` — コンパクト化後のトレースデータバッファ

**使用箇所**: [[ref_lumen_screen_probe_tracing]] — SW RT / HW RT 両トレース前のコンパクト化

---

### GenerateBRDF_PDF

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

| 引数 | 型 | 説明 |
|------|-----|------|
| `SceneTextures` | `const FSceneTextures&` | GBuffer（ラフネス・法線・メタリック）の参照元 |
| `BRDFProbabilityDensityFunction` | `FRDGTextureRef&` | 出力: BRDF PDF テクスチャ（八面体マップ）|
| `BRDFProbabilityDensityFunctionSH` | `FRDGBufferSRVRef&` | 出力: PDF の SH バッファ（Radiance Cache との組み合わせ用）|

**使用箇所**: [[ref_lumen_screen_probe_importance]] — `UseImportanceSampling()` が true の場合に実行

---

### GenerateImportanceSamplingRays

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

**出力**: `ScreenProbeParameters.ImportanceSampling.StructuredImportanceSampledRayInfosForTracing` を更新

**使用箇所**: [[ref_lumen_screen_probe_importance]] — PDF 生成後に呼ばれ、IS レイ方向を構築

---

### TraceScreenProbes

```cpp
void TraceScreenProbes(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    bool bTraceMeshObjects,
    const FSceneTextures& SceneTextures,
    FRDGTextureRef LightingChannelsTexture,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters& RadianceCacheParameters,
    FScreenProbeParameters& ScreenProbeParameters,
    FLumenMeshSDFGridParameters& MeshSDFGridParameters,
    ERDGPassFlags ComputePassFlags);
```

| 引数 | 型 | 説明 |
|------|-----|------|
| `bTraceMeshObjects` | `bool` | Mesh SDF トレースを実行するか |
| `LightingChannelsTexture` | `FRDGTextureRef` | ライティングチャンネルマスク |
| `RadianceCacheParameters` | `FRadianceCacheInterpolationParameters` | ミス時に Radiance Cache から補間 |
| `MeshSDFGridParameters` | `FLumenMeshSDFGridParameters&` | フラスタムグリッドカリング結果 |

**使用箇所**: [[ref_lumen_screen_probe_tracing]] — SW RT パスのエントリポイント実装

---

### RenderHardwareRayTracingScreenProbe

```cpp
void RenderHardwareRayTracingScreenProbe(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FSceneTextureParameters& SceneTextures,
    FScreenProbeParameters& CommonDiffuseParameters,
    const FViewInfo& View,
    const FLumenCardTracingParameters& TracingParameters,
    FLumenIndirectTracingParameters& DiffuseTracingParameters,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters& RadianceCacheParameters,
    ERDGPassFlags ComputePassFlags);
```

**使用箇所**: [[ref_lumen_screen_probe_hwrt]] — HW RT パスのエントリポイント実装

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
