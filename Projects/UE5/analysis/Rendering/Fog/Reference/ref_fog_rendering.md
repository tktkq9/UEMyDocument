# ref: FFogUniformParameters / FExponentialHeightFogSceneInfo / FHeightFogVS

- 対象ファイル: `FogRendering.h/.cpp`
- 概要: [[19_fog_overview]]

---

## FFogUniformParameters（FogRendering.h:16）

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FFogUniformParameters,)
    // ─── 指数霧パラメータ（第1レイヤー）────────────────────────
    SHADER_PARAMETER(FVector4f, ExponentialFogParameters)
    // .x = FogDensity × exp(-FogHeightFalloff × (CameraHeight - FogHeight))
    // .y = FogHeightFalloff（高さ減衰率）
    // .z = FogMaxOpacity（最大不透明度, 0〜1）
    // .w = FogHeight（ワールド Z 座標）

    SHADER_PARAMETER(FVector4f, ExponentialFogParameters2)
    // .x = FogDensity（第2レイヤー）
    // .y = FogDensity（第2レイヤー密度積）
    // .z = FogHeightFalloff（第2レイヤー）
    // .w = FogHeight（第2レイヤー）

    SHADER_PARAMETER(FVector4f, ExponentialFogColorParameter)
    // .xyz = InscatteringColor（霧の色）
    // .w = FogCutoffDistance（霧の終端距離）

    SHADER_PARAMETER(FVector4f, ExponentialFogParameters3)
    // .x = FogStartDistance（霧の開始距離）
    // .yzw = 予約

    SHADER_PARAMETER(FVector4f, SkyAtmosphereAmbientContributionColorScale)
    // Sky Atmosphere のアンビエント霧色への寄与スケール

    // ─── 方向散乱（太陽方向への散乱）───────────────────────────
    SHADER_PARAMETER(FVector4f, InscatteringLightDirection)
    // .xyz = 主ライスト方向（正規化）
    // .w = -DirectionalInscatteringStartDistance（負で無効）

    SHADER_PARAMETER(FVector4f, DirectionalInscatteringColor)
    // .xyz = 方向散乱色（太陽色）
    // .w = DirectionalInscatteringExponent（散乱の鋭さ）

    SHADER_PARAMETER(FVector2f, SinCosInscatteringColorCubemapRotation)
    // キューブマップ散乱色の回転
    SHADER_PARAMETER(FVector3f, FogInscatteringTextureParameters)
    // キューブマップパラメータ
    SHADER_PARAMETER_TEXTURE(TextureCube, FogInscatteringColorCubemap)
    SHADER_PARAMETER_SAMPLER(SamplerState, FogInscatteringColorSampler)

    // ─── 共通 ──────────────────────────────────────────────────
    SHADER_PARAMETER(float, EndDistance)               // 霧の終端距離

    // ─── Volumetric Fog 連携 ──────────────────────────────────
    SHADER_PARAMETER(float, ApplyVolumetricFog)        // 1.0 = Volumetric Fog 使用
    SHADER_PARAMETER(float, VolumetricFogStartDistance) // Volumetric Fog 開始距離
    SHADER_PARAMETER(float, VolumetricFogNearFadeInDistanceInv) // フェードイン逆数
    SHADER_PARAMETER(FVector3f, VolumetricFogAlbedo)   // Volumetric Fog の散乱アルベド
    SHADER_PARAMETER(float, VolumetricFogPhaseG)       // Henyey-Greenstein g 値

    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, IntegratedLightScattering)
    SHADER_PARAMETER_SAMPLER(SamplerState, IntegratedLightScatteringSampler)
    // → Volumetric Fog の積分結果（32×32×16 フラクセル）
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

---

## FExponentialHeightFogSceneInfo（FogRendering.cpp / Engine 側）

```cpp
// UExponentialHeightFogComponent のレンダースレッド側表現
struct FExponentialHeightFogSceneInfo
{
    // ─── 霧パラメータ ─────────────────────────────────────────
    float FogDensity;              // 霧の密度（0〜1）
    float FogHeightFalloff;        // 高さ減衰係数（大きいほど低い位置に霧）
    FLinearColor FogInscatteringColor; // 霧の散乱色（InscatteringColor）
    float FogMaxOpacity;           // 最大不透明度（1.0 = 完全に隠れる）
    float StartDistance;           // 霧の開始距離
    float FogCutoffDistance;       // 霧の終端距離（0 = 無制限）
    float FogHeight;               // 霧の中心高さ（Z 軸）

    // ─── 第2レイヤー ──────────────────────────────────────────
    float SecondFogDensity;
    float SecondFogHeightFalloff;
    float SecondFogHeightOffset;

    // ─── 方向散乱 ─────────────────────────────────────────────
    float DirectionalInscatteringExponent;
    float DirectionalInscatteringStartDistance;
    FLinearColor DirectionalInscatteringColor;

    // ─── Volumetric Fog 設定 ──────────────────────────────────
    bool bEnableVolumetricFog;
    float VolumetricFogScatteringDistribution; // Phase G 値
    FLinearColor VolumetricFogAlbedo;          // 散乱アルベド
    FLinearColor VolumetricFogEmissive;        // 自発光
    float VolumetricFogExtinctionScale;        // 消散スケール
    float VolumetricFogDistance;               // 最大距離
    float VolumetricFogStaticLightingScatteringIntensity;
};
```

---

## FHeightFogVS / FHeightFogPS

```cpp
// FogRendering.cpp（フルスクリーンクワッド描画）

class FHeightFogVS : public FGlobalShader
{
    // FullscreenQuad を描画
    // → ScreenUV と ViewRay を出力
    // → VS_OUT: float4 Position, float2 ScreenUV, float3 ViewRay
};

class FHeightFogPS : public FGlobalShader
{
    // HeightFog.usf でのピクセル処理
    // 入力: SceneDepth, FFogUniformParameters
    // 処理:
    //   WorldPos = ReconstructWorldPos(ScreenUV, SceneDepth)
    //   RayLength = length(WorldPos - CameraPos)
    //
    //   [ApplyVolumetricFog = 0 の場合]
    //   T = ExponentialFog(RayLength, FogHeight, FogDensity, HeightFalloff)
    //   Inscattering = FogColor × (1 - T) + DirectionalInscattering
    //
    //   [ApplyVolumetricFog = 1 の場合]
    //   FroxelUV = TranslatedWorldToFroxelUV(WorldPos)
    //   (Inscattering, T) = IntegratedLightScattering.SampleLevel(FroxelUV, 0)
    //
    //   SceneColor = SceneColor × T + Inscattering
};
```

---

## ユーティリティ関数（FogRendering.h）

```cpp
// FFogUniformParameters を生成してバインド
void SetupFogUniformParameters(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FFogUniformParameters& OutParameters,
    bool bForRealtimeSkyCapture = false);

// UB を生成
TRDGUniformBufferRef<FFogUniformParameters> CreateFogUniformBuffer(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View);

// 霧を描画すべきかどうか判定
bool ShouldRenderFog(const FSceneViewFamily& Family);
// → r.Fog=1 かつ FogDensity > 0

// 霧の共通開始距離を取得（Volumetric / LocalFogVolume を考慮）
float GetViewFogCommonStartDistance(
    const FViewInfo& View,
    bool bShouldRenderVolumetricFog,
    bool bShouldRenderLocalFogVolumes);

float GetFogDefaultStartDistance();
```

---

## 関連リファレンス

- [[ref_volumetric_fog]] — `FVolumetricFogIntegrationParameters` / Froxel 仕様
- [[a_height_fog]] — RenderFog() 詳細フロー
