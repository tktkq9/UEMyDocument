# b: Volumetric Fog（RenderVolumetricFog）

- 対象ファイル: `VolumetricFog.h/.cpp` / `VolumetricFogShared.h`
- 概要: [[19_fog_overview]]

---

## 概要

`RenderVolumetricFog()` は**3D フラクセルグリッド**（Frustum Voxel）に散乱・吸収係数を注入し、  
前方積分して IntegratedLightScattering テクスチャを生成する。  
Height Fog の HeightFogPS はこのテクスチャをサンプルして物理的な光散乱を表現する。

---

## FVolumetricFogIntegrationParameterData（VolumetricFogShared.h）

```cpp
struct FVolumetricFogIntegrationParameterData
{
    bool bTemporalHistoryIsValid; // 時間的再投影の有効フラグ

    TArray<FVector4f, TInlineAllocator<16>> FrameJitterOffsetValues;
    // Froxel ごとのジッタオフセット（時間的ノイズ低減）

    FRDGTexture* VBufferA;       // 散乱係数（RGB) + 消散係数（A）
    FRDGTexture* VBufferB;       // Emissive（RGB）+ Phase G（A）
    FRDGTextureUAV* VBufferA_UAV;
    FRDGTextureUAV* VBufferB_UAV;

    FRDGTexture* LightScattering;     // 光散乱の積分結果
    FRDGTextureUAV* LightScatteringUAV;
};
```

---

## FVolumetricFogIntegrationParameters（VolumetricFogShared.h）

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FVolumetricFogIntegrationParameters, )
    SHADER_PARAMETER_STRUCT_REF(FVolumetricFogGlobalData, VolumetricFog)
    // → GridSize, InvGridSize, GridZParams, FrameJitterOffset 等

    SHADER_PARAMETER(FMatrix44f, UnjitteredClipToTranslatedWorld)
    // Jitter なしの逆投影行列（時間的再投影に必要）

    SHADER_PARAMETER(FMatrix44f, UnjitteredPrevTranslatedWorldToClip)
    // 前フレームの投影行列

    SHADER_PARAMETER_ARRAY(FVector4f, FrameJitterOffsets, [16])
    // 各フラクセルへのサブピクセルジッタ

    SHADER_PARAMETER(float, HistoryWeight)         // 前フレームの重み (0〜1)
    SHADER_PARAMETER(uint32, HistoryMissSuperSampleCount) // 履歴ミス時の追加サンプル
END_SHADER_PARAMETER_STRUCT()
```

---

## Froxel 座標計算

```cpp
// VolumetricFogShared.h:56
inline int32 ComputeZSliceFromDepth(float SceneDepth, FVector GridZParams)
{
    // 対数分布スライス: 近距離は細かく、遠距離は粗く
    return FMath::TruncToInt(
        FMath::Log2(SceneDepth * GridZParams.X + GridZParams.Y) * GridZParams.Z
    );
}

// GridZParams:
//   .X = (FarPlane - NearPlane) / (NearPlane × 2^GridSizeZ)
//   .Y = 1.0
//   .Z = GridSizeZ / log2(FarPlane / NearPlane)

FVector GetVolumetricFogGridZParams(float NearPlane, float FarPlane, int32 GridSizeZ);
```

---

## RenderVolumetricFog() フロー（VolumetricFog.cpp）

```
RenderVolumetricFog(GraphBuilder, SceneTextures, Views, ...)
  │
  ├─ [A] フラクセルグリッド確保（VBufferA/B, LightScattering）
  │   GridSize = (Ceil(ViewWidth/r.VolumetricFog.GridPixelSize),
  │               Ceil(ViewHeight/r.VolumetricFog.GridPixelSize),
  │               r.VolumetricFog.GridSizeZ)
  │   フォーマット: R16G16B16A16F (FP16 × 4)
  │
  ├─ [B] VoxelizeVolumeFog CS（VolumetricFogVoxelization.usf）
  │   入力: ExponentialHeightFogComponent のパラメータ
  │         LocalFogVolume（アクタ配置の局所フォグ）
  │         ParticleSystem のフォグ密度
  │   出力: VBufferA（散乱・消散）, VBufferB（Emissive）
  │
  │   各 Froxel について:
  │     Height Fog: σ_s = FogDensity × exp(-FogHeightFalloff × z)
  │     LocalFogVolume: 範囲内のみ加算
  │     ParticleSystem: SubUV テクスチャ密度から計算
  │
  ├─ [C] InjectLocalLightShadow CS
  │   → Shadow Map（または VSM）からシャドウをサンプル
  │   → 各 Froxel の LightScattering に Local Light の寄与を注入
  │   → Directional Light は InjectLightScattering CS で別途処理
  │
  ├─ [D] IntegrateLightScattering CS（VolumetricFog.usf）
  │   Z 方向に前方（カメラから遠方に向けて）積分:
  │
  │   Transmittance = 1.0
  │   for z = 0 to GridSizeZ:
  │     σ_s = VBufferA[z].rgb   // 散乱係数
  │     σ_e = VBufferA[z].a     // 消散係数
  │     Emissive = VBufferB[z].rgb
  │
  │     // Phase 関数（Henyey-Greenstein）
  │     Phase = HGPhase(cos_theta, PhaseG)
  │
  │     // ライスト寄与
  │     LightColor = LightScatteringBuffer[z]
  │     Inscattering = σ_s × Phase × LightColor + Emissive
  │
  │     // 積分（Beer-Lambert の逆数値化）
  │     dz = FroxelDepthSize
  │     FogAccumulation += Inscattering × Transmittance × dz
  │     Transmittance *= exp(-σ_e × dz)
  │
  │     LightScattering[z] = (FogAccumulation, Transmittance)
  │
  ├─ [E] Temporal Reprojection（bTemporalHistoryIsValid 時）
  │   → 前フレームの LightScattering を UnjitteredPrevTranslatedWorldToClip で再投影
  │   → HistoryWeight でブレンド（デフォルト 0.5）
  │   → 時間的ノイズを低減
  │
  └─ [F] FFogUniformParameters に IntegratedLightScattering を格納
      → HeightFogPS からサンプルされる
```

---

## VolumetricFogTemporalRandom（時間的ジッタ）

```cpp
// VolumetricFogShared.cpp
FVector3f VolumetricFogTemporalRandom(uint32 FrameNumber)
{
    // Halton Sequence で 3 次元低不一致ジッタを生成
    // FrameNumber に基づいてフラクセルをサブサンプル
    // → 各フレームは異なる Froxel 位置をサンプル
    // → 時間的フィルタで平均化することで品質向上
}
```

---

## 関連リファレンス

- [[ref_volumetric_fog]] — `FVolumetricFogIntegrationParameters` / `FVoxelizeVolumePassUniformParameters`
- [[a_height_fog]] — Height Fog（HeightFogPS）フロー
