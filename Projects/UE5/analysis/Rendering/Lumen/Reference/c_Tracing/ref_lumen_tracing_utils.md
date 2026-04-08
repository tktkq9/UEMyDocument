# リファレンス：LumenTracingUtils.h / LumenTracingUtils.cpp

- グループ: c - Tracing
- 上位: [[c_lumen_tracing]]
- 関連: [[ref_lumen_scene_data]] | [[ref_lumen_mesh_sdf_culling]] | [[ref_lumen_heightfields]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenTracingUtils.h/cpp`

---

## 概要

Lumen のトレーシング全体で共用されるシェーダーパラメータ構造体・ユーティリティ関数を定義するヘッダ。  
Surface Cache への Radiance サンプリング、Mesh SDF トレーシング、間接照明トレーシングのパラメータを一元管理する。

---

## FHemisphereDirectionSampleGenerator

半球（または全球）上のサンプル方向を事前生成するユーティリティクラス。  
Radiosity・Screen Probe Gather・Irradiance Field の全方向サンプリングに使われる。

```cpp
class FHemisphereDirectionSampleGenerator {
public:
    TArray<FVector4f> SampleDirections;  // 生成されたサンプル方向リスト
    float ConeHalfAngle = 0;             // 錐体トレース時の半角
    int32 Seed = 0;                      // 乱数シード
    int32 PowerOfTwoDivisor = 1;         // 2の冪乗での分割数
    bool bFullSphere = false;            // 全球サンプリングか（false=半球）
    bool bCosineDistribution = false;    // コサイン分布か（false=均一分布）

    // サンプルを生成してSampleDirectionsを埋める
    void GenerateSamples(int32 TargetNumSamples, int32 InPowerOfTwoDivisor,
                         int32 InSeed, bool bInFullSphere, bool bInCosineDistribution);

    void GetSampleDirections(const FVector4f*& OutDirections, int32& OutNumDirections) const;
};
```

---

## FLumenCardTracingParameters

Surface Cache へのアクセスとフィードバック書き込みに必要な**全シェーダーパラメータ**。  
Lumen のほぼすべてのトレーシングシェーダーが `SHADER_PARAMETER_STRUCT_INCLUDE` でインクルードする。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenCardTracingParameters, )
    // ビュー・シーン
    SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneUniformParameters, Scene)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FReflectionUniformParameters, ReflectionStruct)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FLumenCardScene, LumenCardScene)

    // ライティング設定
    SHADER_PARAMETER(float, DiffuseColorBoost)
    SHADER_PARAMETER(FVector3f, SkylightLeakingColor)
    SHADER_PARAMETER(float, ReflectionSkylightLeakingAverageAlbedo)
    SHADER_PARAMETER(float, SkylightLeakingRoughness)
    SHADER_PARAMETER(float, InvFullSkylightLeakingDistance)
    SHADER_PARAMETER(float, CachedLightingPreExposure)
    SHADER_PARAMETER(float, OneOverCachedLightingPreExposure)

    // フォグ
    SHADER_PARAMETER(uint32, SampleHeightFog)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FFogUniformParameters, FogUniformParameters)

    // Surface Cache フィードバックバッファ（UAV）
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>,   RWCardPageLastUsedBuffer)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>,   RWCardPageHighResLastUsedBuffer)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>,   RWSurfaceCacheFeedbackBufferAllocator)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint2>,  RWSurfaceCacheFeedbackBuffer)
    SHADER_PARAMETER(uint32,    SurfaceCacheFeedbackBufferSize)
    SHADER_PARAMETER(uint32,    SurfaceCacheFeedbackBufferTileWrapMask)
    SHADER_PARAMETER(FIntPoint, SurfaceCacheFeedbackBufferTileJitter)
    SHADER_PARAMETER(float,     SurfaceCacheFeedbackResLevelBias)
    SHADER_PARAMETER(uint32,    SurfaceCacheUpdateFrameIndex)

    // Surface Cache アトラステクスチャ（SRV）
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DirectLightingAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, IndirectLightingAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, FinalLightingAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, AlbedoAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, OpacityAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, NormalAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, EmissiveAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DepthAtlas)

    // タイルシャドウ・Global SDF
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint4>, TileShadowDownsampleFactorAtlasForResampling)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, GlobalDistanceFieldPageObjectGridBuffer)
    SHADER_PARAMETER(uint32, NumGlobalSDFClipmaps)
END_SHADER_PARAMETER_STRUCT()
```

---

## FLumenMeshSDFTracingParameters

Mesh SDF（符号付き距離フィールド）でのトレーシングに必要なパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenMeshSDFTracingParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FDistanceFieldObjectBufferParameters, DistanceFieldObjectBuffers)
    SHADER_PARAMETER_STRUCT_INCLUDE(FDistanceFieldAtlasParameters, DistanceFieldAtlas)
    SHADER_PARAMETER(float, MeshSDFNotCoveredExpandSurfaceScale)   // 両面マテリアル用サーフェス拡張スケール
    SHADER_PARAMETER(float, MeshSDFNotCoveredMinStepScale)         // 最小ステップサイズスケール
    SHADER_PARAMETER(float, MeshSDFDitheredTransparencyStepThreshold) // ディザ透明度スレッショルド
END_SHADER_PARAMETER_STRUCT()
```

---

## FLumenMeshSDFGridParameters

フラスタムグリッドに Mesh SDF オブジェクトをカリングした結果を持つパラメータ。  
`CullMeshObjectsToViewGrid()` の出力として使用される。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenMeshSDFGridParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FLumenMeshSDFTracingParameters, TracingParameters)
    // グリッド内 Mesh SDF
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, NumGridCulledMeshSDFObjects)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, GridCulledMeshSDFObjectStartOffsetArray)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, GridCulledMeshSDFObjectIndicesArray)
    SHADER_PARAMETER(uint32,    CardGridPixelSizeShift)  // グリッドセルのピクセルサイズ（2の冪）
    SHADER_PARAMETER(FVector3f, CardGridZParams)         // Z方向グリッドパラメータ
    SHADER_PARAMETER(FIntVector, CullGridSize)           // グリッドサイズ（XYZ）
    // グリッド内 Heightfield
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, NumCulledHeightfieldObjects)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, CulledHeightfieldObjectIndexBuffer)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, NumGridCulledHeightfieldObjects)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, GridCulledHeightfieldObjectStartOffsetArray)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, GridCulledHeightfieldObjectIndicesArray)
END_SHADER_PARAMETER_STRUCT()
```

---

## FLumenIndirectTracingParameters

間接照明トレーシングの距離・角度・バイアスを制御するパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenIndirectTracingParameters, )
    SHADER_PARAMETER(float, StepFactor)                    // SDF ステップ係数
    SHADER_PARAMETER(float, CardTraceEndDistanceFromCamera) // Surface Cache トレースの最大距離
    SHADER_PARAMETER(float, DiffuseConeHalfAngle)          // 拡散コーンの半角
    SHADER_PARAMETER(float, TanDiffuseConeHalfAngle)       // tan(DiffuseConeHalfAngle)
    SHADER_PARAMETER(float, MinSampleRadius)               // 最小サンプル半径
    SHADER_PARAMETER(float, MinTraceDistance)              // トレース開始距離（自己交差防止）
    SHADER_PARAMETER(float, MaxTraceDistance)              // トレース最大距離
    SHADER_PARAMETER(float, MaxMeshSDFTraceDistance)       // Mesh SDF トレース最大距離
    SHADER_PARAMETER(float, SurfaceBias)                   // サーフェスバイアス
    SHADER_PARAMETER(float, CardInterpolateInfluenceRadius) // Surface Cache 補間影響半径
    SHADER_PARAMETER(float, SpecularFromDiffuseRoughnessStart) // 拡散→鏡面切り替え開始ラフネス
    SHADER_PARAMETER(float, SpecularFromDiffuseRoughnessEnd)   // 拡散→鏡面切り替え終了ラフネス
    SHADER_PARAMETER(int32, HeightfieldMaxTracingSteps)    // Heightfield の最大ステップ数
END_SHADER_PARAMETER_STRUCT()
```

---

## FLumenHZBScreenTraceParameters

HZB（Hierarchical Z Buffer）を使ったスクリーンスペーストレースのパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenHZBScreenTraceParameters, )
    SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture2D, PrevSceneColorTexture)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, HistorySceneDepth)
    SHADER_PARAMETER_STRUCT_INCLUDE(FHZBParameters, HZBParameters)
    SHADER_PARAMETER(FVector2f, PrevSceneColorBilinearUVMin)
    SHADER_PARAMETER(FVector2f, PrevSceneColorBilinearUVMax)
    SHADER_PARAMETER(FVector4f, PrevScreenPositionScaleBias)
    SHADER_PARAMETER(FVector4f, PrevScreenPositionScaleBiasForDepth)
    SHADER_PARAMETER(float, PrevSceneColorPreExposureCorrection)
END_SHADER_PARAMETER_STRUCT()
```

---

## FLumenScreenSpaceBentNormalParameters

Short Range AO / Screen Space Bent Normal を参照するパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenScreenSpaceBentNormalParameters, )
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<UNORM float>,  ShortRangeAOTexture)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float3>,       ShortRangeGITexture)
    SHADER_PARAMETER(FIntPoint, ShortRangeAOViewMin)
    SHADER_PARAMETER(FIntPoint, ShortRangeAOViewSize)
    SHADER_PARAMETER(uint32,    ShortRangeAOMode)
END_SHADER_PARAMETER_STRUCT()
```

---

## 主要ユーティリティ関数

| 関数 | 説明 |
|-----|------|
| `GetLumenCardTracingParameters(GraphBuilder, View, LumenSceneData, FrameTemporaries, bSurfaceCacheFeedback, OutParams)` | FLumenCardTracingParameters を初期化 |
| `CullHeightfieldObjectsForView(...)` | Heightfield オブジェクトをビューに対してカリング |
| `CullMeshObjectsToViewGrid(...)` | Mesh SDF オブジェクトをフラスタムグリッドにカリング |
| `CullForCardTracing(...)` | Card トレーシング用にカリング（IndirectTracingParameters ベース）|
| `SetupLumenDiffuseTracingParameters(MaxTraceDistance, OrthoMaxDimension, OutParams)` | 拡散トレーシングパラメータを設定 |
| `SetupLumenDiffuseTracingParametersForProbe(MaxTraceDistance, OrthoMaxDimension, OutParams, DiffuseConeHalfAngle)` | プローブ用拡散トレーシングパラメータを設定 |
| `SetupLumenMeshSDFTracingParameters(GraphBuilder, Scene, View, OutParams)` | Mesh SDF トレーシングパラメータを設定 |
| `SetupHZBScreenTraceParameters(GraphBuilder, View, SceneTextures)` | HZB スクリーントレースパラメータを設定 |

---

## LumenIrradianceFieldGather 名前空間

```cpp
extern int32 GLumenIrradianceFieldGather; // CVar値（実験的機能の有効フラグ）

namespace LumenIrradianceFieldGather {
    // Irradiance Field 用の RadianceCache 入力パラメータを構築
    LumenRadianceCache::FRadianceCacheInputs SetupRadianceCacheInputs();
}
```

---

## LumenDiffuseIndirect 名前空間

```cpp
namespace LumenDiffuseIndirect {
    bool IsAllowed();  // Diffuse GI が有効かどうか
    bool UseAsyncCompute(const FViewFamilyInfo& ViewFamily, EDiffuseIndirectMethod DiffuseIndirectMethod);
}
```
