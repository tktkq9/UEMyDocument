# リファレンス：LumenTranslucencyVolumeLighting.h/cpp / LumenTranslucencyVolumeHardwareRayTracing.cpp

- グループ: d - Radiance Cache
- 上位: [[d_lumen_radiance_cache]]
- 関連: [[ref_lumen_radiance_cache]] | [[ref_lumen_translucency_radiance_cache]] | [[ref_lumen_hwrt_common]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenTranslucencyVolumeLighting.h`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenTranslucencyVolumeLighting.cpp`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenTranslucencyVolumeHardwareRayTracing.cpp`

---

## 概要

透明マテリアル（特にフォグ・ボリュームトレース）への **GI ボリューム照明** を提供するファイル群。  
フラスタム空間の 3D グリッド（フロクセル）に Lumen の間接照明を焼き込み、  
透明 BasePass や VolumetricFog がサンプリングできるようにする。  
Radiance Cache の補間結果を Volume テクスチャに書き込む形で実装される。

---

## FLumenTranslucencyGIVolume

GI ボリュームテクスチャと Radiance Cache パラメータをまとめたクラス。  
`RenderLumenTranslucencyVolumeLighting()` で生成され、透明 BasePass に渡される。

```cpp
class FLumenTranslucencyGIVolume {
public:
    // Radiance Cache 補間パラメータ（Volume トレースのヒット時照明に使用）
    LumenRadianceCache::FRadianceCacheInterpolationParameters RadianceCacheInterpolationParameters;

    // GI ボリュームテクスチャ（2 本は SH の RGB 成分を分離格納）
    FRDGTextureRef Texture0        = nullptr; // SH 係数セット 0（R, G, B チャンネルまとめ）
    FRDGTextureRef Texture1        = nullptr; // SH 係数セット 1
    FRDGTextureRef HistoryTexture0 = nullptr; // 前フレームのテクスチャ 0（テンポラル蓄積）
    FRDGTextureRef HistoryTexture1 = nullptr; // 前フレームのテクスチャ 1

    // フロクセルグリッドパラメータ
    FVector   GridZParams      = FVector::ZeroVector; // Z 方向グリッドパラメータ
    uint32    GridPixelSizeShift = 0;                 // グリッドセルのピクセルサイズ（2の冪）
    FIntVector GridSize        = FIntVector::ZeroValue; // グリッドサイズ（XYZ）
};
```

---

## FLumenTranslucencyLightingParameters

透明 BasePass シェーダーが参照する照明パラメータ（シェーダー構造体）。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenTranslucencyLightingParameters, )
    // Radiance Cache 補間パラメータ（透明サーフェスの反射照明）
    SHADER_PARAMETER_STRUCT_INCLUDE(LumenRadianceCache::FRadianceCacheInterpolationParameters,
        RadianceCacheInterpolationParameters)

    // Front Layer Translucency Reflection（前面レイヤー専用）
    SHADER_PARAMETER_STRUCT_INCLUDE(FLumenFrontLayerTranslucencyReflectionParameters,
        FrontLayerTranslucencyReflectionParameters)

    // GI ボリュームテクスチャ（間接照明）
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, TranslucencyGIVolume0)
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, TranslucencyGIVolume1)
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, TranslucencyGIVolumeHistory0)
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, TranslucencyGIVolumeHistory1)
    SHADER_PARAMETER_SAMPLER(SamplerState, TranslucencyGIVolumeSampler)

    // フロクセルグリッドのレイアウト
    SHADER_PARAMETER(FVector3f, TranslucencyGIGridZParams)
    SHADER_PARAMETER(uint32,    TranslucencyGIGridPixelSizeShift)
    SHADER_PARAMETER(FIntVector, TranslucencyGIGridSize)
END_SHADER_PARAMETER_STRUCT()

// FLumenTranslucencyGIVolume から上記パラメータを構築するヘルパー
extern FLumenTranslucencyLightingParameters GetLumenTranslucencyLightingParameters(
    FRDGBuilder& GraphBuilder,
    const FLumenTranslucencyGIVolume& LumenTranslucencyGIVolume,
    const FLumenFrontLayerTranslucency& LumenFrontLayerTranslucency);
```

---

## FLumenTranslucencyLightingUniforms

VolumetricFog や HeterogeneousVolumes がバインドするグローバルユニフォームバッファ。

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FLumenTranslucencyLightingUniforms, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FLumenTranslucencyLightingParameters, Parameters)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

---

## FLumenTranslucencyLightingVolumeParameters / TraceSetupParameters

Volume トレーシングに使うパラメータ群。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenTranslucencyLightingVolumeParameters, )
    SHADER_PARAMETER_STRUCT_REF(FBlueNoise, BlueNoise)          // ブルーノイズ（ジッター用）
    SHADER_PARAMETER(FVector3f, TranslucencyGIGridZParams)
    SHADER_PARAMETER(uint32,    TranslucencyGIGridPixelSizeShift)
    SHADER_PARAMETER(FIntVector, TranslucencyGIGridSize)
    SHADER_PARAMETER(int32,  FroxelDirectionJitterFrameIndex)   // フロクセル方向ジッター
    SHADER_PARAMETER(FVector3f, FrameJitterOffset)
    SHADER_PARAMETER(FMatrix44f, UnjitteredClipToTranslatedWorld)
    SHADER_PARAMETER(uint32, TranslucencyVolumeTracingOctahedronResolution) // 八面体マップ解像度
    SHADER_PARAMETER(float, HZBMipLevel)
    SHADER_PARAMETER(float, GridCenterOffsetFromDepthBuffer)
    SHADER_PARAMETER(float, GridCenterOffsetThresholdToAcceptDepthBufferOffset)
    SHADER_PARAMETER_STRUCT_INCLUDE(FHZBParameters, HZBParameters)
END_SHADER_PARAMETER_STRUCT()

BEGIN_SHADER_PARAMETER_STRUCT(FLumenTranslucencyLightingVolumeTraceSetupParameters, )
    SHADER_PARAMETER(float, StepFactor)             // SDF ステップ係数
    SHADER_PARAMETER(float, MaxTraceDistance)        // 最大トレース距離
    SHADER_PARAMETER(float, VoxelTraceStartDistanceScale) // ボクセルトレース開始距離係数
    SHADER_PARAMETER(float, MaxRayIntensity)         // 最大レイ輝度クランプ
END_SHADER_PARAMETER_STRUCT()
```

---

## Lumen 名前空間（Translucency Volume 判定）

```cpp
namespace Lumen {
    // Translucency Volume の HW RT を使うか
    bool UseHardwareRayTracedTranslucencyVolume(const FSceneViewFamily& ViewFamily);

    // 透明マテリアルへの Radiance Cache Reflections を使うか
    bool UseLumenTranslucencyRadianceCacheReflections(const FSceneViewFamily& ViewFamily);
}
```

---

## LumenTranslucencyVolumeRadianceCache 名前空間

```cpp
namespace LumenTranslucencyVolumeRadianceCache {
    // Translucency Volume 用の Radiance Cache 入力パラメータを構築
    LumenRadianceCache::FRadianceCacheInputs SetupRadianceCacheInputs(const FViewInfo& View);
}
```

---

## HW RT エントリポイント（LumenTranslucencyVolumeHardwareRayTracing.cpp）

```cpp
// SW RT の代わりに HW RT でフロクセルをトレースする関数
extern void HardwareRayTraceTranslucencyVolume(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FLumenCardTracingParameters& TracingParameters,
    LumenRadianceCache::FRadianceCacheInterpolationParameters RadianceCacheParameters,
    FLumenTranslucencyLightingVolumeParameters VolumeParameters,
    FLumenTranslucencyLightingVolumeTraceSetupParameters TraceSetupParameters,
    FRDGTextureRef VolumeTraceRadiance,    // 出力: トレースした Radiance
    FRDGTextureRef VolumeTraceHitDistance, // 出力: ヒット距離
    ERDGPassFlags ComputePassFlags);
```

---

## Translucency GI ボリュームの更新フロー

```
RenderLumenTranslucencyVolumeLighting()
  │
  ├─ フロクセルグリッドのサイズ決定
  │   → GridSize / GridZParams / GridPixelSizeShift を計算
  │
  ├─ [Radiance Cache 更新]
  │   → LumenTranslucencyVolumeRadianceCache::SetupRadianceCacheInputs()
  │   → UpdateRadianceCaches() でプローブをトレース
  │
  ├─ [Volume トレース]
  │   ├─ SW RT: 各フロクセルから Surface Cache をトレース
  │   └─ HW RT: HardwareRayTraceTranslucencyVolume() を呼ぶ
  │      → VolumeTraceRadiance / VolumeTraceHitDistance に書き込み
  │
  ├─ [SH への射影]
  │   → フロクセルの Radiance → SH 係数 → Texture0 / Texture1
  │
  ├─ [テンポラル蓄積]
  │   → HistoryTexture0/1 と現フレームを混合してノイズ低減
  │
  └─ FLumenTranslucencyGIVolume を透明 BasePass / VolumetricFog に渡す
```
