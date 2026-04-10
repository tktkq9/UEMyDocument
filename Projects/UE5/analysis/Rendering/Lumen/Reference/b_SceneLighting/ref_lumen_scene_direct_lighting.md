# リファレンス：LumenSceneDirectLighting.cpp / LumenSceneDirectLightingStochastic.inl

- グループ: b - Scene Lighting
- 上位: [[b_lumen_scene_lighting]]
- 関連: [[ref_lumen_scene_lighting]] | [[ref_lumen_scene_direct_lighting_hwrt]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneDirectLighting.cpp`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneDirectLightingStochastic.inl`

---

## 概要

Surface Cache の **Direct Lighting**（直接照明）計算を担うファイル。  
シーン内の全ライトを収集し、Card タイルごとに光源をカリング・シャドウを解決して  
Direct Lighting アトラスに書き込む。  
標準パス（決定論的）と確率的ライティングパス（`Stochastic.inl`）の2系統を持つ。

---

## FLumenGatheredLight

Lumen Direct Lighting で処理する**ライト1灯分のデータ**をまとめたクラス。

```cpp
class FLumenGatheredLight {
public:
    const FLightSceneInfo* LightSceneInfo = nullptr;
    const FMaterialRenderProxy* LightFunctionMaterialProxy;
    uint32 LightIndex = 0;
    ELumenLightType Type;
    bool bHasShadows = false;
    bool bMayCastCloudTransmittance;
    bool bNeedsShadowMask = false;
    bool bBatchedShadowsEligible;
    FString Name;

    TArray<TUniformBufferRef<FDeferredLightUniformStruct>,
           TInlineAllocator<4>> DeferredLightUniformBuffers;

    bool NeedsShadowMask() const;
    bool CanUseBatchedShadows() const;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `LightSceneInfo` | `const FLightSceneInfo*` | ライトのシーン情報（位置・色・範囲など）|
| `LightFunctionMaterialProxy` | `const FMaterialRenderProxy*` | ライト関数マテリアル（null なら使用しない）|
| `LightIndex` | `uint32` | `GatheredLights` 配列内のインデックス（GPU シェーダーへ渡す）|
| `Type` | `ELumenLightType` | Directional / Point / Spot / Rect |
| `bHasShadows` | `bool` | ダイナミックシャドウを落とすか |
| `bMayCastCloudTransmittance` | `bool` | 雲シャドウサンプリングが必要か |
| `bNeedsShadowMask` | `bool` | シャドウマスクバッファへの書き込みが必要か |
| `bBatchedShadowsEligible` | `bool` | バッチシャドウ処理が可能か（Directional 以外かつシャドウあり）|
| `Name` | `FString` | デバッグ用ライト名 |
| `DeferredLightUniformBuffers` | `TArray<TUniformBufferRef<...>, TInlineAllocator<4>>` | ビューオリジンごとの Deferred Light UB（マルチビュー対応）|

### NeedsShadowMask

```cpp
bool FLumenGatheredLight::NeedsShadowMask() const {
    return bHasShadows || bMayCastCloudTransmittance;
}
```

### CanUseBatchedShadows

```cpp
bool FLumenGatheredLight::CanUseBatchedShadows() const {
    return bBatchedShadowsEligible;
    // Directional ライトは個別処理（ライト関数・雲シャドウ）のため除外
}
```

### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `GatherLights()` — 生成
- [[ref_lumen_scene_direct_lighting]] `BuildLightTiles()` — ライトタイルカリングに参照
- [[ref_lumen_scene_direct_lighting_hwrt]] `TraceLumenHardwareRayTracedDirectLightingShadows()` — HW RT シャドウトレースに渡す

---

## FLumenLightTileScatterParameters

ライトタイルの Indirect Dispatch に使うシェーダーパラメータ構造体。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenLightTileScatterParameters, )
    RDG_BUFFER_ACCESS(DrawIndirectArgs, ERHIAccess::IndirectArgs)
    RDG_BUFFER_ACCESS(DispatchIndirectArgs, ERHIAccess::IndirectArgs)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>,  LightTileAllocator)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint2>, LightTiles)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>,  LightTileOffsetsPerLight)
    SHADER_PARAMETER(int32, bUseLightTilesPerLightType)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `DrawIndirectArgs` | `FRDGBufferRef` | DrawIndirect 用引数バッファ |
| `DispatchIndirectArgs` | `FRDGBufferRef` | DispatchIndirect 用引数バッファ |
| `LightTileAllocator` | `StructuredBuffer<uint>` | ライトタイル数カウンタ |
| `LightTiles` | `StructuredBuffer<uint2>` | ライトタイルデータ（CardTileIndex + LightIndex）|
| `LightTileOffsetsPerLight` | `StructuredBuffer<uint>` | ライトタイプ別のタイル開始オフセット |
| `bUseLightTilesPerLightType` | `int32` | ライトタイプ別にタイルを分けるか（`UseLightTilesPerLightType()` の結果）|

### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `BuildLightTiles()` — 生成・書き込み
- [[ref_lumen_scene_direct_lighting]] `TraceShadows()` — Dispatch 間接引数として参照
- [[ref_lumen_scene_direct_lighting_hwrt]] `TraceLumenHardwareRayTracedDirectLightingShadows()` — HW RT パスへ渡す

---

## LumenSceneDirectLighting 名前空間

### シャドウレイバイアス関数

各トレースメソッドごとに独立したバイアス値を返す。  
`FMath::Max(..., 0.0f)` でゼロ未満を防ぐ。

| 関数 | CVar | デフォルト |
|-----|------|----------|
| `GetMeshSDFShadowRayBias()` | `r.LumenScene.DirectLighting.MeshSDF.ShadowRayBias` | 2.0 |
| `GetHeightfieldShadowRayBias()` | `r.LumenScene.DirectLighting.Heightfield.ShadowRayBias` | 2.0 |
| `GetGlobalSDFShadowRayBias()` | `r.LumenScene.DirectLighting.GlobalSDF.ShadowRayBias` | 1.0 |
| `GetHardwareRayTracingShadowRayBias()` | `r.LumenScene.DirectLighting.HardwareRayTracing.ShadowRayBias` | 1.0 |

### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `FLumenSceneDirectLightingTraceDistanceFieldShadowsCS` パラメータ設定
- [[ref_lumen_scene_direct_lighting_hwrt]] `SetLumenHardwareRayTracedDirectLightingShadowsParameters()`

---

### 判定関数

```cpp
// バッチシャドウ: r.LumenScene.DirectLighting.BatchShadows == 2 でライトタイプごとにタイル分割
bool UseLightTilesPerLightType();

// 確率的ライティングを使うか（Screen Probe GI との連携）
bool UseStochasticLighting(const FSceneViewFamily& ViewFamily);

// HW RT Direct Lighting の有効判定
bool Lumen::UseHardwareRayTracedDirectLighting(const FSceneViewFamily&);

// Far Field を Direct Lighting で使うか
bool UseFarField(const FSceneViewFamily& ViewFamily);

// 全面を両面としてトレースするか
bool IsForceTwoSided();
```

| 関数 | 条件 |
|-----|------|
| `UseLightTilesPerLightType()` | `r.LumenScene.DirectLighting.BatchShadows == 2` |
| `UseStochasticLighting()` | `LumenScreenProbeGather` が有効かつ CVar > 0 |
| `UseHardwareRayTracedDirectLighting()` | `RHI_RAYTRACING && IsRayTracingEnabled() && CVar != 0` |
| `UseFarField()` | `Lumen::UseFarField() && CVar != 0` |
| `IsForceTwoSided()` | `r.LumenScene.DirectLighting.HardwareRayTracing.ForceTwoSided != 0` |

---

## 主要シェーダークラス

### FBuildLightTilesCS

Card タイルごとにどのライトが影響するか計算する CS。

```cpp
class FBuildLightTilesCS : public FGlobalShader {
    class FMaxLightSamples : SHADER_PERMUTATION_SPARSE_INT("MAX_LIGHT_SAMPLES", 1,2,4,8,16,32);
    using FPermutationDomain = TShaderPermutationDomain<FMaxLightSamples>;

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>, RWLightTileAllocator)
        SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint2>, RWLightTiles)
        SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>, RWLightTileAllocatorPerLight)
        SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint2>, CardTiles)
        SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, CardTileDepthRanges)
        SHADER_PARAMETER(int32, CullToCardTileDepthRange)
        SHADER_PARAMETER(int32, MaxLightsPerTile)
        SHADER_PARAMETER(uint32, NumLights)
        SHADER_PARAMETER(FMatrix44f, FrustumTranslatedWorldToClip)
        SHADER_PARAMETER(FVector3f, PreViewTranslation)
        SHADER_PARAMETER(float, ViewExposure)
        SHADER_PARAMETER(int32, bUseLightTilesPerLightType)
    END_SHADER_PARAMETER_STRUCT()
};
```

#### パーミュテーション

| パーミュテーション | 値 | 説明 |
|--------------------|-----|------|
| `FMaxLightSamples` | 1/2/4/8/16/32 | タイルあたりの最大サンプル数（= MaxLightsPerTile）|

#### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `BuildLightTiles()` で実行

---

### FLumenDirectLightingHardwareRayTracing

HW RT によるシャドウトレース CS / RGS。

```cpp
class FLumenDirectLightingHardwareRayTracing : public FLumenHardwareRayTracingShaderBase {
    class FForceTwoSided                 : SHADER_PERMUTATION_BOOL("FORCE_TWO_SIDED");
    class FEnableFarFieldTracing         : SHADER_PERMUTATION_BOOL("FAR_FIELD_TRACING");
    class FEnableHeightfieldProjectionBias : SHADER_PERMUTATION_BOOL("HEIGHTFIELD_PROJECTION_BIAS");
    class FSurfaceCacheAlphaMasking      : SHADER_PERMUTATION_BOOL("ALPHA_MASKING");
    class FStochastic                    : SHADER_PERMUTATION_BOOL("STOCHASTIC");
    using FPermutationDomain = TShaderPermutationDomain<
        FLumenHardwareRayTracingShaderBase::FBasePermutationDomain,
        FForceTwoSided, FEnableFarFieldTracing,
        FEnableHeightfieldProjectionBias, FSurfaceCacheAlphaMasking, FStochastic>;
};
```

#### メンバ変数（FParameters）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `HardwareRayTracingIndirectArgs` | `FRDGBufferRef` | Dispatch Indirect 引数 |
| `LightTileAllocator/LightTiles` | `SRV` | ライトタイルバッファ |
| `LumenLightData` | `FLumenLightDataParameters` | ライトデータ |
| `ShadowTraceAllocator/ShadowTraces` | `SRV` | シャドウトレースバッファ |
| `PullbackBias` | `float` | 自己遮蔽防止のプルバックバイアス |
| `MaxTraceDistance` | `float` | 最大トレース距離 |
| `HardwareRayTracingShadowRayBias` | `float` | シャドウレイバイアス |
| `RWShadowMaskTiles` | `UAV` | 出力: シャドウマスクタイル |
| `FrustumTranslatedWorldToClip` | `FMatrix44f` | Stochastic 用フラスタム行列 |
| `RWLightSamples` | `UAV` | Stochastic 用出力ライトサンプル |
| `CompactedLightSampleData/Allocator` | `SRV` | Stochastic 用コンパクトデータ |

#### 使用箇所
- [[ref_lumen_scene_direct_lighting_hwrt]] `TraceLumenHardwareRayTracedDirectLightingShadows()` から実行

---

### FLumenSceneDirectLightingTraceDistanceFieldShadowsCS

SDF（Signed Distance Field）を使ってシャドウをトレースする CS。

```cpp
class FLumenSceneDirectLightingTraceDistanceFieldShadowsCS : public FGlobalShader {
    class FThreadGroupSize32         : SHADER_PERMUTATION_BOOL("THREADGROUP_SIZE_32");
    class FLightType                 : SHADER_PERMUTATION_ENUM_CLASS("LIGHT_TYPE", ELumenLightType);
    class FTraceGlobalSDF            : SHADER_PERMUTATION_BOOL("TRACE_GLOBAL_SDF");
    class FSimpleCoverageBasedExpand : SHADER_PERMUTATION_BOOL("SIMPLE_COVERAGE_BASED_EXPAND");
    class FTraceMeshSDFs             : SHADER_PERMUTATION_BOOL("TRACE_MESH_SDFS");
    class FTraceHeightfields         : SHADER_PERMUTATION_BOOL("TRACE_HEIGHTFIELDS");
    class FOffsetDataStructure       : SHADER_PERMUTATION_RANGE_INT("OFFSET_DATA_STRUCTURE", 0, 2);
    using FPermutationDomain = TShaderPermutationDomain<
        FThreadGroupSize32, FLightType, FTraceGlobalSDF,
        FSimpleCoverageBasedExpand, FTraceMeshSDFs, FTraceHeightfields, FOffsetDataStructure>;
};
```

#### 主要パラメータ

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `MeshSDFShadowRayBias` | `float` | Mesh SDF シャドウのバイアス |
| `HeightfieldShadowRayBias` | `float` | Heightfield シャドウのバイアス |
| `GlobalSDFShadowRayBias` | `float` | Global SDF シャドウのバイアス |
| `StepFactor` | `float` | SDF マーチングのステップ係数 |
| `TranslatedWorldToShadow` | `FMatrix44f` | World → Shadow 変換行列 |
| `ObjectBufferParameters` | `...` | Mesh SDF オブジェクトバッファ |
| `LightTileIntersectionParameters` | `...` | ライトタイル交差バッファ |

---

## 処理関数

### ClearLumenSceneDirectLighting

```cpp
void ClearLumenSceneDirectLighting(
    FRDGBuilder& GraphBuilder,
    const FLumenCardUpdateContext& CardUpdateContext,
    const FLumenSceneFrameTemporaries& FrameTemporaries);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `CardUpdateContext` | `const FLumenCardUpdateContext&` | 更新対象ページの Indirect Args |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | Direct Lighting アトラス |

#### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `RenderDirectLighting()` の最初に呼ばれる

#### 内部処理フロー

```cpp
void ClearLumenSceneDirectLighting(...)
{
    // FRasterizeToCardsVS + FClearLumenCardsPS(NumTargets=1 or 2) でアトラスをクリア
    // DrawQuadsToAtlas<FClearLumenCardsPS> を使用
    auto* PassParams = GraphBuilder.AllocParameters<FClearLumenCardsPS::FParameters>();
    PassParams->View         = View.ViewUniformBuffer;
    PassParams->LumenCardScene = FrameTemporaries.LumenCardSceneUniformBuffer;
    // Indirect Draw で更新対象ページのみクリア
    DrawQuadsToAtlas<FClearLumenCardsPS>(..., PassParams, CardUpdateContext.DrawCardPageIndicesIndirectArgs);
}
```

---

### SetupLightFunctionParameters

```cpp
void SetupLightFunctionParameters(
    const FViewInfo& View,
    const FLumenGatheredLight& GatheredLight,
    float ShadowFadeFraction,
    FLightFunctionParameters& OutParams);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `View` | `const FViewInfo&` | ビュー情報 |
| `GatheredLight` | `const FLumenGatheredLight&` | 対象ライト |
| `ShadowFadeFraction` | `float` | シャドウフェード割合 |
| `OutParams` | `FLightFunctionParameters&` | 出力: ライト関数パラメータ |

#### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `RenderDirectLightingStandard()` — ライト関数あり時

#### 内部処理フロー

```cpp
OutParams.LightFunctionTranslatedWorldToLight = InverseTransform(LightToWorld);
OutParams.OuterConeAngle  = GatheredLight.LightSceneInfo->Proxy->GetOuterConeAngle();
OutParams.CameraRelativeLightPosition = ...;
OutParams.ShadowFadeFraction = ShadowFadeFraction;
OutParams.DisabledBrightness = CVarLumen...;
OutParams.FadeDistance = CVarLumen...;
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.DirectLighting` | 1 | Surface Cache の Direct Lighting の有効/無効 |
| `r.LumenScene.DirectLighting.MaxLightsPerTile` | 8 | タイルあたりの最大ライト数（4/8/16/32）|
| `r.LumenScene.DirectLighting.CullToTileDepthRange` | 1 | タイルの深度範囲でライトをカリングするか |
| `r.LumenScene.DirectLighting.OffscreenShadowing.TraceMeshSDFs` | 1 | オフスクリーンシャドウに Mesh SDF を使うか（0 = Global SDF）|
| `r.LumenScene.DirectLighting.OffscreenShadowingTraceStepFactor` | 5.0 | オフスクリーンシャドウのトレースステップ係数 |
| `r.LumenScene.DirectLighting.CloudTransmittance` | 1 | 雲シャドウのサンプリングを行うか |
| `r.LumenScene.DirectLighting.BatchShadows` | 2 | シャドウパスのバッチ処理（0=無効, 1=タイプ共通, 2=タイプ別）|
| `r.LumenScene.DirectLighting.HardwareRayTracing.AdaptiveShadowTracing` | 1 | 前フレームで均一シャドウだったタイルのレイ数を削減 |

---

## Direct Lighting パイプライン全体フロー

```
RenderLumenSceneLighting()
  │
  └─ RenderDirectLighting(CardUpdateContext)
        │
        ├─ ClearLumenSceneDirectLighting()   ← Direct Lighting アトラスをクリア
        │
        ├─ GatherLights()                    ← シーン内の有効ライトを FLumenGatheredLight に収集
        │   ├─ Directional, Point, Spot, Rect を ELumenLightType で分類
        │   └─ SetDirectLightingDeferredLightUniformBuffer() で各ビュー UB を生成
        │
        ├─ FBuildLightTilesCS               ← Card タイルごとにどのライトが影響するか計算
        │   ├─ FCalculateCardTileDepthRangesCS → タイルの深度範囲計算
        │   ├─ CullToTileDepthRange → タイル深度範囲でタイトなカリング
        │   └─ FComputeLightTileOffsetsPerLightCS → ライトタイプ別オフセット計算
        │
        ├─ [UseStochasticLighting == true]
        │   └─ RenderDirectLightingStochastic()
        │         → 1 Card タイルあたり N サンプルでランダムライト選択
        │         → CompactedLightSampleData に書き込み
        │         → HW RT 連携: LightSamples テクスチャ経由
        │
        ├─ [UseStochasticLighting == false]
        │   └─ RenderDirectLightingStandard()
        │         → ライトタイプ別にシャドウレイを発射
        │         → UseHardwareRayTracedDirectLighting() ? HW RT : SDF
        │         → [HW RT] TraceLumenHardwareRayTracedDirectLightingShadows()
        │         → [SDF]   FLumenSceneDirectLightingTraceDistanceFieldShadowsCS
        │
        └─ FLumenCardBatchDirectLightingCS   → Direct Lighting アトラスに合成
```

---

## バッチ処理可能ライトタイプ

```cpp
// Point / Spot / Rect の 3 種がバッチ対象
// Directional は雲シャドウ・ライト関数の可能性があるため個別処理
constexpr uint32 NumBatchableLightTypes = 3;
```
