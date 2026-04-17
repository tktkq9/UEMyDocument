# ref: FVolumetricFogIntegrationParameters / FVolumetricFogGlobalData

- 対象ファイル: `VolumetricFogShared.h` / `VolumetricFog.h`
- 概要: [[19_fog_overview]]

---

## FVoxelizeVolumePassUniformParameters（VolumetricFogShared.h:9）

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FVoxelizeVolumePassUniformParameters, )
    SHADER_PARAMETER_STRUCT(FSceneTextureUniformParameters, SceneTextures)
    SHADER_PARAMETER(FMatrix44f, ViewToVolumeClip)       // ビュー → ボリューム Clip
    SHADER_PARAMETER(FVector2f, ClipRatio)               // アスペクト比補正
    SHADER_PARAMETER(FVector4f, FrameJitterOffset0)      // 時間的ジッタ
    SHADER_PARAMETER_STRUCT(FVolumetricFogGlobalData, VolumetricFog)

    // Volumetric Cloud 連携パラメータ
    SHADER_PARAMETER(FVector3f, RenderVolumetricCloudParametersCloudLayerCenterKm)
    SHADER_PARAMETER(float, RenderVolumetricCloudParametersPlanetRadiusKm)
    SHADER_PARAMETER(float, RenderVolumetricCloudParametersBottomRadiusKm)
    SHADER_PARAMETER(float, RenderVolumetricCloudParametersTopRadiusKm)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

---

## FVolumetricFogGlobalData（VolumetricFog.h）

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FVolumetricFogGlobalData, )
    // ─── グリッドサイズ ──────────────────────────────────────────
    SHADER_PARAMETER(FIntVector, GridSize)          // (Wx/px, Wy/px, GridSizeZ)
    SHADER_PARAMETER(FVector3f, GridZParams)        // Z スライス対数パラメータ
    // GridZParams.x = (FarPlane - NearPlane) / (NearPlane × 2^GridSizeZ)
    // GridZParams.y = 1.0
    // GridZParams.z = GridSizeZ / log2(FarPlane / NearPlane)

    SHADER_PARAMETER(FVector2f, SVPosToVolumeUV)    // SV_Position → Volume UV
    SHADER_PARAMETER(float, MaxDistance)            // Volumetric Fog の最大距離
    SHADER_PARAMETER(FVector4f, UVScaleBias)        // UV スケール/バイアス

    // ─── 時間的再投影 ────────────────────────────────────────────
    SHADER_PARAMETER(FVector3f, FrameJitterOffset)  // 現フレームのジッタ
    SHADER_PARAMETER(float, HistoryAlpha)           // 前フレームのブレンド率
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

---

## FVolumetricFogIntegrationParameterData（VolumetricFogShared.h:26）

```cpp
struct FVolumetricFogIntegrationParameterData
{
    bool bTemporalHistoryIsValid;   // 有効な時間的履歴があるか

    TArray<FVector4f, TInlineAllocator<16>> FrameJitterOffsetValues;
    // 最大 16 フレーム分のジッタオフセット（Halton 数列ベース）

    // ─── Froxel ボリュームバッファ ─────────────────────────────
    FRDGTexture* VBufferA;          // R16G16B16A16F: (散乱, 消散)
    FRDGTexture* VBufferB;          // R16G16B16A16F: (Emissive.rgb, PhaseG)
    FRDGTextureUAV* VBufferA_UAV;
    FRDGTextureUAV* VBufferB_UAV;

    FRDGTexture* LightScattering;   // 積分済みライットスキャッタリング
    FRDGTextureUAV* LightScatteringUAV;
};
```

---

## FVolumetricFogIntegrationParameters（VolumetricFogShared.h:42）

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FVolumetricFogIntegrationParameters, )
    SHADER_PARAMETER_STRUCT_REF(FVolumetricFogGlobalData, VolumetricFog)

    // 時間的再投影に必要な変換行列
    SHADER_PARAMETER(FMatrix44f, UnjitteredClipToTranslatedWorld)
    // → 現フレームの Jitter なし ClipPos → WorldPos 変換

    SHADER_PARAMETER(FMatrix44f, UnjitteredPrevTranslatedWorldToClip)
    // → 前フレームの Jitter なし WorldPos → ClipPos 変換

    SHADER_PARAMETER_ARRAY(FVector4f, FrameJitterOffsets, [16])
    // サブピクセルジッタ（各フラクセルが少しずつずれた位置をサンプル）

    SHADER_PARAMETER(float, HistoryWeight)              // 前フレームの重み（0〜1）
    SHADER_PARAMETER(uint32, HistoryMissSuperSampleCount) // 履歴ミス時の追加サンプル数
END_SHADER_PARAMETER_STRUCT()
```

---

## Froxel 座標変換ユーティリティ（VolumetricFogShared.h）

```cpp
// Z スライスインデックス計算（対数分布）
inline int32 ComputeZSliceFromDepth(float SceneDepth, FVector GridZParams)
{
    return FMath::TruncToInt(
        FMath::Log2(SceneDepth * GridZParams.X + GridZParams.Y) * GridZParams.Z);
}
// SceneDepth が 2 倍になるたびにスライスが 1 増える（等対数分布）

// GridZParams の計算
FVector GetVolumetricFogGridZParams(
    float NearPlane,
    float FarPlane,
    int32 GridSizeZ);

// 時間的ランダムジッタ（Halton 数列）
FVector3f VolumetricFogTemporalRandom(uint32 FrameNumber);
// → FrameNumber % 16 の Halton 値を返す
// → x: Halton(2), y: Halton(3), z: Halton(5)

// ライトソフトフェーディング強度
float GetVolumetricFogLightSoftFading();
```

---

## Froxel データフォーマット

| バッファ | フォーマット | 内容 |
|--------|------------|------|
| VBufferA | R16G16B16A16F | RGB = 散乱係数(σ_s), A = 消散係数(σ_e) |
| VBufferB | R16G16B16A16F | RGB = Emissive, A = Henyey-Greenstein g |
| LightScattering | R16G16B16A16F | RGB = 積分済み散乱色, A = 積分済み透過率 |

---

## Local Fog Volume との連携

```
LocalFogVolume（ブループリントで配置可能な局所フォグ）:
  → VoxelizeVolumeFog CS で Froxel 範囲内の VBufferA に密度を注入
  → ボックス/球/カプセル形状でブレンド
  → LocalFogVolume 外の Froxel は ExponentialHeightFog の密度のみ
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.VolumetricFog` | 1 | Volumetric Fog 有効 |
| `r.VolumetricFog.GridPixelSize` | 8 | Froxel の XY ピクセルサイズ |
| `r.VolumetricFog.GridSizeZ` | 64 | Z スライス数 |
| `r.VolumetricFog.Distance` | 6000 | 最大距離 [cm] |
| `r.VolumetricFog.TemporalReprojection` | 1 | 時間的再投影有効 |
| `r.VolumetricFog.HistoryMissSupersampleCount` | 4 | 履歴ミス時追加サンプル |

---

## 関連リファレンス

- [[ref_fog_rendering]] — `FFogUniformParameters` / `FHeightFogVS`
- [[b_volumetric_fog]] — RenderVolumetricFog() 詳細フロー
