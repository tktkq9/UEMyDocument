# ref: MegaLights サンプリング パラメータ

- 対象: `MegaLightsInternal.h` / `MegaLightsSampling.cpp`
- Details: [[b_megalights_sampling]]

---

## FMegaLightsParameters（メインシェーダーパラメータ構造体）

```cpp
// MegaLightsInternal.h:11
BEGIN_SHADER_PARAMETER_STRUCT(FMegaLightsParameters, )
    // ビュー・シーン
    SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, ViewUniformBuffer)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FHairStrandsViewUniformParameters, HairStrands)
    SHADER_PARAMETER_STRUCT_INCLUDE(ShaderPrint::FShaderParameters, ShaderPrintUniformBuffer)
    SHADER_PARAMETER_STRUCT_INCLUDE(FSceneTextureParameters, SceneTextures)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneUniformParameters, Scene)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneTextureUniformParameters, SceneTexturesStruct)

    // マテリアル・ライスト
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSubstrateGlobalUniformParameters, Substrate)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FForwardLightUniformParameters, ForwardLightStruct)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(LightFunctionAtlas::FLightFunctionAtlasGlobalParameters, LightFunctionAtlas)
    SHADER_PARAMETER_STRUCT_INCLUDE(FSceneLightingChannelParameters, LightingChannelParameters)

    // サンプリング
    SHADER_PARAMETER_STRUCT_REF(FBlueNoise, BlueNoise)     // Blue Noise テクスチャ（時空間ノイズ低減）
    SHADER_PARAMETER_TEXTURE(Texture2D, PreIntegratedGF)   // IBL スペキュラー LUT
    SHADER_PARAMETER_SAMPLER(SamplerState, PreIntegratedGFSampler)

    // ---- ビューポート・ダウンサンプルパラメータ ----
    SHADER_PARAMETER(FIntPoint, SampleViewMin)             // サンプルバッファ上の View 開始座標
    SHADER_PARAMETER(FIntPoint, SampleViewSize)            // サンプルバッファ上の View サイズ
    SHADER_PARAMETER(FIntPoint, DownsampledViewMin)        // ダウンサンプル後 View 開始座標
    SHADER_PARAMETER(FIntPoint, DownsampledViewSize)       // ダウンサンプル後 View サイズ
    SHADER_PARAMETER(FIntPoint, NumSamplesPerPixel)        // ピクセルあたりサンプル数（2D）
    SHADER_PARAMETER(FIntPoint, NumSamplesPerPixelDivideShift) // 割り算をビットシフトに変換
    SHADER_PARAMETER(FVector2f, DownsampledBufferInvSize)  // バッファ逆サイズ（UV 計算用）
    SHADER_PARAMETER(FIntPoint, DownsampleFactor)          // ダウンサンプル係数

    // ---- フレーム管理 ----
    SHADER_PARAMETER(uint32, MegaLightsStateFrameIndex)         // テンポラルジッタ用フレームインデックス
    SHADER_PARAMETER(uint32, StochasticLightingStateFrameIndex) // Stochastic Lighting 共有フレームカウンタ

    // ---- 重み・閾値 ----
    SHADER_PARAMETER(float, MinSampleWeight)    // サンプル最小重み（これ以下は棄却）
    SHADER_PARAMETER(float, MaxShadingWeight)   // シェーディング最大重み（クランプ）
    SHADER_PARAMETER(int32, TileDataStride)     // タイルデータバッファのストライド
    SHADER_PARAMETER(int32, DownsampledTileDataStride)

    // ---- デバッグ ----
    SHADER_PARAMETER(int32, DebugMode)
    SHADER_PARAMETER(FIntPoint, DebugCursorPosition)
    SHADER_PARAMETER(int32, DebugLightId)
    SHADER_PARAMETER(int32, DebugVisualizeLight)

    // ---- 機能フラグ ----
    SHADER_PARAMETER(int32, UseIESProfiles)        // IES プロファイル使用
    SHADER_PARAMETER(int32, UseLightFunctionAtlas) // Light Function Atlas 使用

    // ---- 行列（アンジッタ済み）----
    SHADER_PARAMETER(FMatrix44f, UnjitteredClipToTranslatedWorld)
    SHADER_PARAMETER(FMatrix44f, UnjitteredTranslatedWorldToClip)
    SHADER_PARAMETER(FMatrix44f, UnjitteredPrevTranslatedWorldToClip)

    // ---- HZB（遮蔽カリング）----
    SHADER_PARAMETER_STRUCT_INCLUDE(FHZBParameters, HZBParameters)

    // ---- ライストハッシュ（テンポラル管理）----
    SHADER_PARAMETER(FIntPoint, VisibleLightHashViewMinInTiles)
    SHADER_PARAMETER(FIntPoint, VisibleLightHashViewSizeInTiles)

    // ---- ダウンサンプル済みテクスチャ ----
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float>, DownsampledSceneDepth)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<UNORM float3>, DownsampledSceneWorldNormal)
END_SHADER_PARAMETER_STRUCT()
```

---

## FMegaLightsVolumeParameters（TranslucencyVolume 用パラメータ）

```cpp
// MegaLightsInternal.h:55
BEGIN_SHADER_PARAMETER_STRUCT(FMegaLightsVolumeParameters, )
    SHADER_PARAMETER(float, VolumeMinSampleWeight)
    SHADER_PARAMETER(float, VolumeMaxShadingWeight)
    SHADER_PARAMETER(int32, VolumeDownsampleFactorMultShift)
    SHADER_PARAMETER(int32, VolumeDebugMode)
    SHADER_PARAMETER(int32, VolumeDebugSliceIndex)

    // ボクセルあたりサンプル数（3D）
    SHADER_PARAMETER(FIntVector, NumSamplesPerVoxel)
    SHADER_PARAMETER(FIntVector, NumSamplesPerVoxelDivideShift)

    // ボリュームグリッドサイズ
    SHADER_PARAMETER(FIntVector, DownsampledVolumeViewSize)
    SHADER_PARAMETER(FIntVector, VolumeViewSize)
    SHADER_PARAMETER(FIntVector, VolumeSampleViewSize)
    SHADER_PARAMETER(FVector3f, VolumeInvBufferSize)

    // Z スライスパラメータ（対数分布）
    SHADER_PARAMETER(FVector3f, MegaLightsVolumeZParams)
    SHADER_PARAMETER(uint32, MegaLightsVolumePixelSize)

    // ボリュームジッタ・フェーズ関数
    SHADER_PARAMETER(FVector3f, VolumeFrameJitterOffset)
    SHADER_PARAMETER(float, VolumePhaseG)                // Henyey-Greenstein 非対称パラメータ

    SHADER_PARAMETER(float, VolumeInverseSquaredLightDistanceBiasScale)
    SHADER_PARAMETER(float, LightSoftFading)

    // TranslucencyVolume カスケード
    SHADER_PARAMETER(uint32, TranslucencyVolumeCascadeIndex)
    SHADER_PARAMETER(float, TranslucencyVolumeInvResolution)

    SHADER_PARAMETER(uint32, UseHZBOcclusionTest)
    SHADER_PARAMETER(uint32, IsUnifiedVolume)

    // リサンプルパラメータ
    SHADER_PARAMETER(FIntVector, ResampleVolumeViewSize)
    SHADER_PARAMETER(FVector3f, ResampleVolumeInvBufferSize)
    SHADER_PARAMETER(FVector3f, ResampleVolumeZParams)
END_SHADER_PARAMETER_STRUCT()
```

---

## EMegaLightsInput（入力タイプ）

```cpp
// MegaLightsInternal.h:82
enum class EMegaLightsInput : uint8
{
    GBuffer,      // 通常の不透明サーフェス（GBuffer デコード）
    HairStrands,  // Hair Strands サンプル（独立シェーディングパス）
    Count
};
```

---

## RayTraceLightSamples() シグネチャ

```cpp
// MegaLightsInternal.h:92
namespace MegaLights
{
    void RayTraceLightSamples(
        const FSceneViewFamily& ViewFamily,
        const FViewInfo& View, int32 ViewIndex,
        FRDGBuilder& GraphBuilder,
        const FSceneTextures& SceneTextures,
        const FVirtualShadowMapArray* VirtualShadowMapArray,
        const TArrayView<FRDGTextureRef> NaniteShadingMasks,
        const FIntPoint SampleBufferSize,
        FRDGTextureRef LightSamples,        // 入力: サンプルされたライストインデックス
        FRDGTextureRef LightSampleRays,     // 入力: レイ方向・距離
        // TranslucencyVolume 用
        FIntVector VolumeSampleBufferSize,
        FRDGTextureRef VolumeLightSamples,
        FRDGTextureRef VolumeLightSampleRays,
        FIntVector TranslucencyVolumeSampleBufferSize,
        TArrayView<FRDGTextureRef> TranslucencyVolumeLightSamples,
        TArrayView<FRDGTextureRef> TranslucencyVolumeLightSampleRays,
        // パラメータ
        const FMegaLightsParameters& MegaLightsParameters,
        const FMegaLightsVolumeParameters& MegaLightsVolumeParameters,
        const FMegaLightsVolumeParameters& MegaLightsTranslucencyVolumeParameters,
        EMegaLightsInput InputType,
        bool bDebug);
}
```

---

> [!note]- NumSamplesPerPixel の決め方
> `MegaLights::GetNumSamplesPerPixel2d(InputType)` で計算される。
> `r.MegaLights.NumSamplesPerPixel`（デフォルト 2）を基に、
> DownsampleFactor（デフォルト 2x2=4）と合わせて実効サンプル数が決まる。
> 実際は 4 DownsampledPixels × 2 Samples = 8 サンプル/フルレゾ ピクセル 相当。

> [!note]- UnjitteredXxx 行列の役割
> TSR/TAA のジッタが入った通常の ViewProj 行列とは別に、ジッタなしの行列を保持する。
> MegaLights のリプロジェクションはジッタ補正を自前で行うため、アンジッタ済み行列が必要。

> [!note]- BlueNoise の利用
> `FBlueNoise` は 128×128 のブルーノイズテクスチャをタイル状に配置した構造体。
> 時空間でノイズパターンが偏らないようサンプルジッタに使われる。
> フレームインデックスによって参照するスライスが変わる（テンポラル Blue Noise）。
