# リファレンス：LumenRadiosity.h / LumenRadiosity.cpp

- グループ: b - Scene Lighting
- 上位: [[b_lumen_scene_lighting]]
- 関連: [[ref_lumen_scene_lighting]] | [[ref_lumen_radiance_cache]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenRadiosity.h/cpp`

---

## 概要

Surface Cache 間の**マルチバウンス拡散 GI**（間接照明）を計算するシステム。  
各 Card 表面に均等間隔で**放射照度プローブ**（Radiosity Probe）を配置し、  
Surface Cache の Radiance をトレースして SH（球面調和関数）に焼き付ける。  
計算結果は Indirect Lighting アトラスに書き込まれる。

---

## LumenRadiosity 名前空間

### FFrameTemporaries

1 フレームの Radiosity 計算に必要な一時データを保持する構造体。  
`InitFrameTemporaries()` で初期化され、レンダリング全体に渡される。

```cpp
namespace LumenRadiosity {
    struct FFrameTemporaries {
        bool bIndirectLightingHistoryValid;
        bool bUseProbeOcclusion;

        int32 ProbeSpacing;
        int32 HemisphereProbeResolution;

        FIntPoint ProbeAtlasSize;
        FIntPoint ProbeTracingAtlasSize;

        FRDGTextureRef TraceRadianceAtlas;
        FRDGTextureRef TraceHitDistanceAtlas;

        FRDGTextureRef ProbeSHRedAtlas;
        FRDGTextureRef ProbeSHGreenAtlas;
        FRDGTextureRef ProbeSHBlueAtlas;
    };
}
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `bIndirectLightingHistoryValid` | `bool` | 前フレームの SH アトラスが有効か（テンポラル蓄積の可否）|
| `bUseProbeOcclusion` | `bool` | プローブ補間時に深度オクルージョンを使うか |
| `ProbeSpacing` | `int32` | プローブ間隔（Surface Cache テクセル単位、通常 4）|
| `HemisphereProbeResolution` | `int32` | 半球1辺のトレース数（`N×N` 方向にトレース）|
| `ProbeAtlasSize` | `FIntPoint` | SH 係数アトラスのサイズ（PhysicalAtlasSize / ProbeSpacing × HemisphereProbeResolution²）|
| `ProbeTracingAtlasSize` | `FIntPoint` | トレース結果アトラスのサイズ |
| `TraceRadianceAtlas` | `FRDGTextureRef` | トレースした放射輝度（HDR）のアトラス |
| `TraceHitDistanceAtlas` | `FRDGTextureRef` | トレースのヒット距離アトラス（プローブオクルージョン用）|
| `ProbeSHRedAtlas` | `FRDGTextureRef` | SH 係数アトラス（R チャンネル）|
| `ProbeSHGreenAtlas` | `FRDGTextureRef` | SH 係数アトラス（G チャンネル）|
| `ProbeSHBlueAtlas` | `FRDGTextureRef` | SH 係数アトラス（B チャンネル）|

### 使用箇所
- [[ref_lumen_scene_rendering]] `RenderLumenSceneLighting()` — `LumenRadiosity::InitFrameTemporaries()` で生成・渡される
- [[ref_lumen_scene_lighting]] `FCopyCardCaptureLightingToAtlasPS` — Radiosity アトラスを参照

---

## IsEnabled

```cpp
bool LumenRadiosity::IsEnabled(const FSceneViewFamily& ViewFamily);
```

### 戻り値
`bool` — Radiosity が有効かどうか

### 内部動作
```cpp
bool LumenRadiosity::IsEnabled(const FSceneViewFamily& ViewFamily) {
    return GLumenRadiosity != 0
        && ViewFamily.EngineShowFlags.LumenSecondaryBounces;
}
```

### 使用箇所
- [[ref_lumen_scene_rendering]] `RenderLumenSceneLighting()` — Radiosity パスを実行するか判定

---

## InitFrameTemporaries

```cpp
void LumenRadiosity::InitFrameTemporaries(
    FRDGBuilder& GraphBuilder,
    const FLumenSceneFrameTemporaries& LumenFrameTemporaries,
    const FViewInfo& View,
    FFrameTemporaries& OutFrameTemporaries);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `LumenFrameTemporaries` | `const FLumenSceneFrameTemporaries&` | 物理アトラスサイズなど |
| `View` | `const FViewInfo&` | ビュー情報（品質設定参照）|
| `OutFrameTemporaries` | `FFrameTemporaries&` | 出力: 初期化されたフレームデータ |

### 使用箇所
- [[ref_lumen_scene_rendering]] `RenderLumenSceneLighting()` の冒頭

### 内部処理フロー

1. **プローブ設定の計算**
   ```cpp
   if (IsEnabled(ViewFamily) && bFinalLightingAtlasContentsValid) {
       OutFrameTemporaries.ProbeSpacing = GetRadiosityProbeSpacing(View);
       // LumenSceneLightingQuality >= 6 で半減、RoundUpToPowerOfTwo + Clamp(1, CardTileSize)
       
       OutFrameTemporaries.HemisphereProbeResolution = GetHemisphereProbeResolution(View);
       // LumenSceneLightingQuality に応じて変化、Clamp(1, 16)
   }
   ```

2. **プローブオクルージョンの判定**
   ```cpp
   OutFrameTemporaries.bUseProbeOcclusion = UseProbeOcclusion();
   // GRadiosityFilteringProbeOcclusion != 0 && Strength > 0
   ```

3. **アトラスサイズの計算**
   ```cpp
   const FIntPoint PhysicalAtlasSize = LumenData.GetPhysicalAtlasSize();
   const uint32 DownsampleFactor = GetAtlasDownsampleFactor(); // = 1
   OutFrameTemporaries.ProbeAtlasSize        = PhysicalAtlasSize / DownsampleFactor;
   OutFrameTemporaries.ProbeTracingAtlasSize = PhysicalAtlasSize / DownsampleFactor;
   ```

4. **RDG テクスチャの登録または生成**
   ```cpp
   OutFrameTemporaries.TraceRadianceAtlas = RegisterOrCreateRadiosityAtlas(
       GraphBuilder, TraceRadianceAtlasTexture, ProbeTracingAtlasSize, PF_FloatRGB, ...);
   
   if (bUseProbeOcclusion) {
       OutFrameTemporaries.TraceHitDistanceAtlas = RegisterOrCreateRadiosityAtlas(...);
   }
   
   OutFrameTemporaries.ProbeSHRedAtlas   = RegisterOrCreateRadiosityAtlas(...ProbeSHRedAtlas...);
   OutFrameTemporaries.ProbeSHGreenAtlas = RegisterOrCreateRadiosityAtlas(...);
   OutFrameTemporaries.ProbeSHBlueAtlas  = RegisterOrCreateRadiosityAtlas(...);
   ```

---

## GetAtlasDownsampleFactor

```cpp
uint32 LumenRadiosity::GetAtlasDownsampleFactor();
```

### 戻り値
`uint32` — 常に 1（現在はダウンサンプルなし）

### 使用箇所
- [[ref_lumen_radiosity]] `InitFrameTemporaries()` — アトラスサイズ計算

---

## 主要シェーダークラス

### FBuildRadiosityTilesCS

Card ページから Radiosity タイルを構築する CS。

```cpp
class FBuildRadiosityTilesCS : public FGlobalShader {
    // パラメータ:
    //   IndirectArgBuffer, LumenCardScene
    //   RadiosityCommonParameters, RadiosityTexelTraceParameters
    //   RWCardTileAllocator, RWCardTileData
    //   RWRadiosityTileAllocator, RWRadiosityTileData
    //   CardPageIndexAllocator, CardPageIndexData
    //   FrustumTranslatedWorldToClip, PreViewTranslation
    // ThreadGroup: 8
};
```

#### 使用箇所
- [[ref_lumen_radiosity]] `LumenRadiosity::AddRadiosityPass()` — タイル構築ステップ

---

### FLumenRadiosityDistanceFieldTracingCS

SDF を使って Radiosity プローブをトレースする CS。

```cpp
class FLumenRadiosityDistanceFieldTracingCS : public FGlobalShader {
    class FThreadGroupSize32         : SHADER_PERMUTATION_BOOL("THREADGROUP_SIZE_32");
    class FTraceGlobalSDF            : SHADER_PERMUTATION_BOOL("TRACE_GLOBAL_SDF");
    class FSimpleCoverageBasedExpand : SHADER_PERMUTATION_BOOL("SIMPLE_COVERAGE_BASED_EXPAND");
    class FProbeOcclusion            : SHADER_PERMUTATION_BOOL("PROBE_OCCLUSION");
    using FPermutationDomain = TShaderPermutationDomain<
        FThreadGroupSize32, FTraceGlobalSDF, FSimpleCoverageBasedExpand, FProbeOcclusion>;
    // ModifyCompilationEnvironment: ENABLE_DYNAMIC_SKY_LIGHT=1, CFLAG_Wave32
};
```

#### 主要パラメータ

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `IndirectArgs` | `FRDGBufferRef` | Dispatch 間接引数 |
| `RadiosityTexelTraceParameters` | `struct` | プローブ設定・アトラス参照 |
| `TracingParameters` | `FLumenCardTracingParameters` | Card トレーシング共通パラメータ |
| `MaxRayIntensity` | `float` | 輝度クランプ値 |
| `RWTraceRadianceAtlas` | `UAV` | 出力: トレース放射輝度 |
| `RWTraceHitDistanceAtlas` | `UAV` | 出力: ヒット距離（オクルージョン用）|

#### 使用箇所
- [[ref_lumen_radiosity]] `LumenRadiosity::AddRadiosityPass()` — SDF トレースパス

---

> [!note]- FLumenRadiosityHardwareRayTracing — HW RT プローブトレース
> 
> ```cpp
> class FLumenRadiosityHardwareRayTracing : public FLumenHardwareRayTracingShaderBase {
>     class FAvoidSelfIntersectionsMode : SHADER_PERMUTATION_RANGE_INT(..., 0, 1);
>     class FSurfaceCacheAlphaMasking   : SHADER_PERMUTATION_BOOL(...);
>     class FProbeOcclusion             : SHADER_PERMUTATION_BOOL(...);
>     // ModifyCompilationEnvironment: ENABLE_DYNAMIC_SKY_LIGHT=1
>     // ERayTracingPayloadType::LumenMinimal
> };
> ```
> 
> **パラメータ**: `SharedParameters, HardwareRayTracingIndirectArgs, RadiosityTexelTraceParameters, NumThreadsToDispatch, MinTraceDistance, MaxTraceDistance, SurfaceBias, MaxRayIntensity, RWTraceRadianceAtlas, RWTraceHitDistanceAtlas`
> 
> **使用箇所**: [[ref_lumen_radiosity]] `AddRadiosityPass()` — `Lumen::UseHardwareRayTracedRadiosity()` が true の場合

---

> [!note]- FLumenRadiositySpatialFilterProbeRadiance — 空間フィルタリング
> 
> ```cpp
> class FLumenRadiositySpatialFilterProbeRadiance : public FGlobalShader {
>     class FPlaneWeighting  : SHADER_PERMUTATION_BOOL("PLANE_WEIGHTING");
>     class FProbeOcclusion  : SHADER_PERMUTATION_BOOL("PROBE_OCCLUSION");
>     class FKernelSize      : SHADER_PERMUTATION_RANGE_INT("KERNEL_SIZE", 0, 2);
> };
> ```
> 
> **説明**: 近傍プローブの TraceRadiance を空間フィルタリングしてノイズを削減。平面重み付けにより裏面プローブの影響を抑える。
> 
> **パラメータ**: `IndirectArgs, RadiosityTexelTraceParameters, RWFilteredTraceRadianceAtlas, ProbePlaneWeightingDepthScale`
> 
> **使用箇所**: [[ref_lumen_radiosity]] `AddRadiosityPass()` — トレース後のフィルタリングステップ

---

> [!note]- FLumenRadiosityConvertToSH — SH 変換
> 
> ```cpp
> class FLumenRadiosityConvertToSH : public FGlobalShader {
>     // ThreadGroup: 64
> };
> ```
> 
> **説明**: TraceRadiance（半球方向の輝度）を L1 SH 係数（4係数 × RGB = 12 値）に変換。
> 
> **パラメータ**: `RWRadiosityProbeSHRedAtlas, RWRadiosityProbeSHGreenAtlas, RWRadiosityProbeSHBlueAtlas, RadiosityTexelTraceParameters, IndirectArgs`
> 
> **使用箇所**: [[ref_lumen_radiosity]] `AddRadiosityPass()` — フィルタリング後の SH 変換ステップ

---

> [!note]- FLumenRadiosityIntegrateCS — SH から Indirect Lighting を生成
> 
> ```cpp
> class FLumenRadiosityIntegrateCS : public FGlobalShader {
>     class FPlaneWeighting       : SHADER_PERMUTATION_BOOL("PLANE_WEIGHTING");
>     class FProbeOcclusion       : SHADER_PERMUTATION_BOOL("PROBE_OCCLUSION");
>     class FTemporalAccumulation : SHADER_PERMUTATION_BOOL("TEMPORAL_ACCUMULATION");
>     // ThreadGroup: 8, CompilerFlags: AllowTypedUAVLoads
> };
> ```
> 
> **説明**: SH 係数を Card テクセルに補間・展開して Indirect Lighting アトラスに書き込む。  
> テンポラル蓄積が有効な場合、前フレームのアトラスと加重平均する。
> 
> **パラメータ**: `RWRadiosityAtlas, RWRadiosityNumFramesAccumulatedAtlas, RadiosityProbeSHRedAtlas/Green/BlueAtlas, ProbePlaneWeightingDepthScale`
> 
> **使用箇所**: [[ref_lumen_radiosity]] `AddRadiosityPass()` — 最終ステップ（Indirect Lighting Atlas への書き込み）

---

## FLumenRadiosityTexelTraceParameters

Radiosity シェーダー全体で共有されるプローブ設定のシェーダーパラメータ構造体。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenRadiosityTexelTraceParameters, )
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>, CardTileAllocator)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>, CardTileData)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>, RadiosityTileAllocator)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint2>, RadiosityTileData)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float3>, TraceRadianceAtlas)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float>, TraceHitDistanceAtlas)
    SHADER_PARAMETER(FIntPoint, RadiosityAtlasSize)
    SHADER_PARAMETER(int32, ProbeSpacingInCardTexels)
    SHADER_PARAMETER(int32, ProbeSpacingInCardTexelsDivideShift)
    SHADER_PARAMETER(int32, CardTileSizeInProbes)
    SHADER_PARAMETER(int32, RadiosityTileSizeInProbes)
    SHADER_PARAMETER(int32, HemisphereProbeResolution)
    SHADER_PARAMETER(int32, NumTracesPerProbe)
    SHADER_PARAMETER(float, ProbeOcclusionStrength)
    SHADER_PARAMETER(int32, FixedJitterIndex)
    SHADER_PARAMETER(int32, MaxFramesAccumulated)
    SHADER_PARAMETER(int32, NumViews)
    SHADER_PARAMETER(int32, ViewIndex)
    SHADER_PARAMETER(int32, MaxRadiosityTiles)
    SHADER_PARAMETER(float, TargetFormatQuantizationError)
    SHADER_PARAMETER_STRUCT_INCLUDE(FBlueNoise, BlueNoise)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `CardTileAllocator/Data` | `SRV` | Card タイルバッファ |
| `RadiosityTileAllocator/Data` | `SRV` | Radiosity タイルバッファ |
| `TraceRadianceAtlas` | `UAV` | トレース放射輝度アトラス（書き込み用）|
| `TraceHitDistanceAtlas` | `UAV` | ヒット距離アトラス（書き込み用、オクルージョン時のみ）|
| `ProbeSpacingInCardTexels` | `int32` | Card テクセル単位のプローブ間隔 |
| `ProbeSpacingInCardTexelsDivideShift` | `int32` | プローブ間隔の除算を bit shift で行うためのシフト量 |
| `CardTileSizeInProbes` | `int32` | Card タイル（8×8 px）内のプローブ数 |
| `HemisphereProbeResolution` | `int32` | 半球の解像度（N で N×N 方向にトレース）|
| `NumTracesPerProbe` | `int32` | プローブあたりのトレース数（= HemisphereProbeResolution²）|
| `ProbeOcclusionStrength` | `float` | オクルージョン強度（0〜1）|
| `FixedJitterIndex` | `int32` | 固定ジッターインデックス（-1 でランダム）|
| `MaxFramesAccumulated` | `int32` | テンポラル蓄積の最大フレーム数 |
| `BlueNoise` | `FBlueNoise` | ブルーノイズテクスチャ参照 |

---

## 主要 CVar（LumenRadiosity.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.Radiosity` | 1 | Radiosity の有効/無効（0 で無効化）|
| `r.LumenScene.Radiosity.ProbeSpacing` | 4 | プローブ間隔（Surface Cache テクセル単位）|
| `r.LumenScene.Radiosity.HemisphereProbeResolution` | 4 | 半球1辺あたりのトレース数（4×4 = 16 レイ）|
| `r.LumenScene.Radiosity.SpatialFilterProbes` | 1 | プローブの空間フィルタリングを行うか |
| `r.LumenScene.Radiosity.SpatialFilterProbes.KernelSize` | 1 | フィルタカーネルサイズ（0/1/2）|
| `r.LumenScene.Radiosity.ProbePlaneWeighting` | 1 | 平面距離による重みづけでリークを抑制 |
| `r.LumenScene.Radiosity.ProbeOcclusion` | 0 | プローブ補間時に深度テストするか |
| `r.LumenScene.Radiosity.ProbeOcclusionStrength` | 0.5 | プローブオクルージョン強度（0〜1）|
| `r.LumenScene.Radiosity.SpatialFilterProbes.PlaneWeightingDepthScale` | -100.0 | 平面重みの深度スケール |
| `r.LumenScene.Radiosity.MaxRayIntensity` | 40.0 | トレース輝度クランプ値（露光相対）|
| `r.LumenScene.Radiosity.DistanceFieldSurfaceBias` | 10.0 | SDF トレースの表面バイアス（cm）|
| `r.LumenScene.Radiosity.DistanceFieldSurfaceSlopeBias` | 5.0 | SDF トレースの法線方向バイアス |
| `r.LumenScene.Radiosity.HardwareRayTracing.SurfaceBias` | 0.1 | HW RT トレースの表面バイアス |
| `r.LumenScene.Radiosity.HardwareRayTracing.SurfaceSlopeBias` | 0.2 | HW RT トレースの法線方向バイアス |
| `r.LumenScene.Radiosity.UpdateFactor` | 64 | 更新レート（全ページの 1/64 を毎フレーム更新）|
| `r.LumenScene.Radiosity.TemporalAccumulation` | 1 | テンポラル蓄積を使うか |
| `r.LumenScene.Radiosity.MaxFramesAccumulated` | 4 | テンポラル蓄積の最大フレーム数 |
| `r.LumenScene.Radiosity.FixedJitterIndex` | -1 | 固定ジッターインデックス（-1=ランダム）|
| `r.LumenScene.Radiosity.HardwareRayTracing` | 1 | HW RT を Radiosity に使用するか |

---

## Radiosity プローブの仕組み

### プローブ配置

```
Surface Cache Card (例: 128×128 テクセル)
  ProbeSpacing = 4 の場合:
  → 32×32 個のプローブを均等配置
  → 各プローブは半球 4×4 = 16 方向にトレース
```

### SH（球面調和関数）への変換

```
各プローブのトレース結果（Radiance × 方向）
  → L1 SH（4係数 × RGB = 12 値）に射影
  → ProbeSHRed / Green / Blue アトラスに保存
  → 補間時に Card テクセルの法線方向で SH を評価 → Irradiance
```

### ERadiosityIndirectArgs

```cpp
enum ERadiosityIndirectArgs {
    NumTracesDiv64 = 0,               // ÷64 スレッドグループ
    NumTracesDiv32 = 1,               // ÷32 スレッドグループ（Wave32 用）
    ThreadPerProbe = 2,               // 1 スレッド = 1 プローブ
    HardwareRayTracingThreadPerTrace = 3, // HW RT: 1 スレッド = 1 トレース
    MAX = 4
};
```

---

## Radiosity 処理フロー

```
LumenRadiosity::AddRadiosityPass(CardUpdateContext)
  │
  ├─ RadiosityTileSize の決定
  │   └─ ProbeSpacing >= 4 ? ProbeSpacing : CardTileSize
  │
  ├─ バッファの生成
  │   ├─ CardTileAllocator/Tiles, RadiosityTileAllocator/Tiles
  │   └─ DispatchCardTilesIndirectArgs, RadiosityIndirectArgs (ERadiosityIndirectArgs)
  │
  ├─ FBuildRadiosityTilesCS         ← CardPageIndexから RadiosityTiles構築
  ├─ FLumenRadiosityIndirectArgsCS  ← DispatchIndirectArgs設定
  │
  ├─ [トレースパス]
  │   ├─ [SDF]   FLumenRadiosityDistanceFieldTracingCS × ViewOrigins
  │   └─ [HW RT] FLumenRadiosityHardwareRayTracingRGS/CS × ViewOrigins
  │        → TraceRadianceAtlas, TraceHitDistanceAtlas に書き込み
  │
  ├─ [空間フィルタリング] (GLumenRadiositySpatialFilterProbes != 0)
  │   └─ FLumenRadiositySpatialFilterProbeRadiance
  │        → FilteredTraceRadianceAtlas に書き込み
  │
  ├─ FLumenRadiosityConvertToSH     ← TraceRadiance → SH係数
  │        → ProbeSHRed/Green/BlueAtlas に書き込み
  │
  └─ FLumenRadiosityIntegrateCS     ← SH → Indirect Lighting Atlas
       ├─ テンポラル蓄積 (UseTemporalAccumulation())
       └─ → IndirectLightingAtlas に書き込み
```

---

## UpdateFactor の意味

```cpp
// r.LumenScene.Radiosity.UpdateFactor = 64 の場合:
// 全 Card ページのうち 1/64 ずつを毎フレーム更新
// 64 フレームで全ページが更新される（1フレームの負荷を分散）
//
// Direct Lighting UpdateFactor = 32 より遅い理由:
// Radiosity はバウンス光なので Direct Lighting より変化が緩やか
```
