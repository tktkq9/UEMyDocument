# ref: FSkyAtmosphereRenderContext / FAtmosphereUniformShaderParameters

- 対象ファイル: `SkyAtmosphereRendering.h/.cpp`
- 概要: [[18_skyatmosphere_overview]]

---

## FAtmosphereUniformShaderParameters（SkyAtmosphereRendering.h:38）

大気の物理パラメータを保持する UniformBuffer。  
`USkyAtmosphereComponent` が変更されると再生成される（静的キャッシュ）。

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FAtmosphereUniformShaderParameters, )
    // ─── 多重散乱 ────────────────────────────────────────────────
    SHADER_PARAMETER(float, MultiScatteringFactor)      // 多重散乱の強度（0〜1）

    // ─── 大気ジオメトリ ──────────────────────────────────────────
    SHADER_PARAMETER(float, BottomRadiusKm)             // 地球表面半径 [km]（6360 km デフォルト）
    SHADER_PARAMETER(float, TopRadiusKm)                // 大気上端半径 [km]（6460 km デフォルト）

    // ─── Rayleigh 散乱（空の青さ）──────────────────────────────
    SHADER_PARAMETER(float, RayleighDensityExpScale)    // 指数密度スケール（= -1/8 km）
    SHADER_PARAMETER(FLinearColor, RayleighScattering)  // 散乱係数 σ_R [1/km] (RGB)

    // ─── Mie 散乱（霞・雲）─────────────────────────────────────
    SHADER_PARAMETER(FLinearColor, MieScattering)       // σ_M [1/km]
    SHADER_PARAMETER(float, MieDensityExpScale)         // Mie 密度指数スケール
    SHADER_PARAMETER(FLinearColor, MieExtinction)       // Mie 消散係数 σ_T [1/km]
    SHADER_PARAMETER(float, MiePhaseG)                  // Henyey-Greenstein g（-1〜1）
    SHADER_PARAMETER(FLinearColor, MieAbsorption)

    // ─── Ozone 層吸収（成層圏の青紫吸収）──────────────────────
    SHADER_PARAMETER(float, AbsorptionDensity0LayerWidth)   // 層の幅 [km]
    SHADER_PARAMETER(float, AbsorptionDensity0ConstantTerm) // 下層の定数項
    SHADER_PARAMETER(float, AbsorptionDensity0LinearTerm)   // 下層の線形項
    SHADER_PARAMETER(float, AbsorptionDensity1ConstantTerm) // 上層の定数項
    SHADER_PARAMETER(float, AbsorptionDensity1LinearTerm)   // 上層の線形項
    SHADER_PARAMETER(FLinearColor, AbsorptionExtinction)    // Ozone 消散係数

    // ─── 地表 ────────────────────────────────────────────────
    SHADER_PARAMETER(FLinearColor, GroundAlbedo)            // 地表アルベド（地面の反射率）
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

---

## FSkyAtmosphereViewSharedUniformShaderParameters（SkyAtmosphereRendering.h:59）

カメラ位置に依存する動的パラメータ（View Global UB に含まれる）。

```cpp
struct FSkyAtmosphereViewSharedUniformShaderParameters
{
    FVector4f CameraAerialPerspectiveVolumeSizeAndInvSize;
    float AerialPerspectiveStartDepthKm;               // 大気効果の開始深度 [km]
    float CameraAerialPerspectiveVolumeDepthResolution; // Volume の Z 解像度
    float CameraAerialPerspectiveVolumeDepthResolutionInv;
    float CameraAerialPerspectiveVolumeDepthSliceLengthKm;    // Z スライス厚 [km]
    float CameraAerialPerspectiveVolumeDepthSliceLengthKmInv;
    float ApplyCameraAerialPerspectiveVolume; // 1.0 = Volume を使用
};
```

---

## FSkyAtmosphereRenderContext（SkyAtmosphereRendering.h:72）

```cpp
struct FSkyAtmosphereRenderContext
{
    // ─── 描画制御フラグ ──────────────────────────────────────────
    bool bUseDepthBoundTestIfPossible; // DepthBoundsTest で空部分のみ描画（最適化）
    bool bForceRayMarching;            // LUT バイパス（最高品質・低速）
    bool bDepthReadDisabled;           // SceneDepth 不使用（リフレクションキャプチャ用）
    bool bDisableBlending;             // ブレンドなし（Sky Capture 用）
    bool bFastSky;                     // SkyViewLUT のみ（高速近似）
    bool bFastAerialPerspective;       // AerialPerspectiveVolume 不使用
    bool bFastAerialPerspectiveDepthTest;
    bool bSecondAtmosphereLightEnabled; // 第二光源（月など）有効
    bool bShouldSampleOpaqueShadow;    // シャドウマップをサンプル

    // ─── LUT テクスチャ ──────────────────────────────────────────
    FRDGTextureRef TransmittanceLut;                    // 256×64
    FRDGTextureRef MultiScatteredLuminanceLut;          // 32×32
    FRDGTextureRef SkyAtmosphereViewLutTexture;         // 192×108（SkyViewLUT）
    FRDGTextureRef SkyAtmosphereCameraAerialPerspectiveVolume;          // 32×32×16
    FRDGTextureRef SkyAtmosphereCameraAerialPerspectiveVolumeMieOnly;
    FRDGTextureRef SkyAtmosphereCameraAerialPerspectiveVolumeRayOnly;

    // ─── ビューパラメータ ────────────────────────────────────────
    FViewMatrices* ViewMatrices;
    TUniformBufferRef<FViewUniformShaderParameters> ViewUniformBuffer;
    TRDGUniformBufferRef<FSceneUniformParameters> SceneUniformBuffer;
    bool bSceneHasSkyMaterial;         // Sky Material を持つメッシュがあるか
};
```

---

## FSkyAtmosphereRenderSceneInfo（SkyAtmosphereRendering.h）

シーン内の `USkyAtmosphereComponent` のレンダースレッド側表現。

```cpp
class FSkyAtmosphereRenderSceneInfo
{
    // FAtmosphereUniformShaderParameters を保持
    // キャッシュ済み LUT テクスチャへの参照
    // TransmittanceLUT / MultiScatteredLUT は変化時のみ更新

    // SkyAtmosphere の可視性チェック・LUT の有効性管理
    bool IsTransmittanceLutCacheValid() const;
    void SetTransmittanceLutCacheDirty();
};
```

---

## LUT キャッシュの更新条件

| LUT | 更新タイミング |
|-----|--------------|
| TransmittanceLUT | 大気パラメータが変化した場合のみ（通常は毎フレーム再利用）|
| MultiScatteredLuminanceLUT | 大気パラメータ or 太陽方向変化時 |
| SkyViewLUT | 毎フレーム（カメラ位置で変わる）|
| AerialPerspectiveVolume | 毎フレーム（カメラ近傍の大気）|

---

## 関連リファレンス

- [[ref_sky_light]] — `FSkyLightSceneProxy` / `FReflectionUniformParameters`
- [[a_sky_atmosphere]] — LUT 生成・RenderSky() フロー詳細
