# a: Exponential Height Fog（RenderFog）

- 対象ファイル: `FogRendering.h/.cpp`
- 概要: [[19_fog_overview]]

---

## 概要

`RenderFog()` は **指数関数的高さ霧** を SceneDepth から WorldPos を再構築して  
ピクセルシェーダーで計算し、SceneColor に乗算合成する。  
Volumetric Fog が有効な場合は IntegratedLightScattering テクスチャを利用して  
物理的な光散乱に置き換えられる。

---

## 指数関数的 Height Fog の計算式

```
// 指数関数的高さ霧の透過率
T = exp(-FogDensity × RayLength × exp(-FogHeightFalloff × (WorldZ - FogHeight)))

パラメータ:
  FogDensity    : 霧の密度（UExponentialHeightFogComponent.FogDensity）
  FogHeightFalloff: 高さによる密度の減衰率
  FogHeight     : 霧の基準高さ
  RayLength     : カメラからフラグメントまでの距離

// 霧の色
FogColor = InscatteringColor + 方向散乱色 × DirectionalInscattering
InscatteringColor: アンビエント霧色
DirectionalInscattering: ライスト方向への散乱（太陽の光等）
```

---

## FFogUniformParameters（FogRendering.h）

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FFogUniformParameters,)
    // 霧パラメータ Pack
    SHADER_PARAMETER(FVector4f, ExponentialFogParameters)
    // .x = FogDensity × HeightFalloff 係数
    // .y = FogHeightFalloff
    // .z = FogMaxOpacity
    // .w = FogHeight（ワールド空間）

    SHADER_PARAMETER(FVector4f, ExponentialFogParameters2)
    // .x = FogDensity（第1フォグレイヤー）
    // .y = FogDensity（第2フォグレイヤー、任意）
    // .z = FogDensity × HeightFalloff（第2）
    // .w = FogHeight（第2）

    SHADER_PARAMETER(FVector4f, ExponentialFogColorParameter)
    // .xyz = InscatteringColor
    // .w = FogCutoffDistance

    SHADER_PARAMETER(FVector4f, ExponentialFogParameters3)
    // .x = FogStartDistance
    // .y = ...

    SHADER_PARAMETER(FVector4f, InscatteringLightDirection)
    // .xyz = 主ライスト方向（太陽）
    // .w = DirectionalInscatteringStartDistance（負の値で無効）

    SHADER_PARAMETER(FVector4f, DirectionalInscatteringColor)
    // .xyz = 方向散乱色
    // .w = DirectionalInscatteringExponent

    // Volumetric Fog 連携
    SHADER_PARAMETER(float, ApplyVolumetricFog)              // 1.0 = Volumetric Fog 使用
    SHADER_PARAMETER(float, VolumetricFogStartDistance)
    SHADER_PARAMETER(float, VolumetricFogNearFadeInDistanceInv)
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, IntegratedLightScattering)
    SHADER_PARAMETER_SAMPLER(SamplerState, IntegratedLightScatteringSampler)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

---

## RenderFog() フロー

```
SetupFogUniformParameters(GraphBuilder, View, OutParameters)
  → ExponentialHeightFogComponent から全パラメータを UB に詰める
  → SkyAtmosphere のアンビエント寄与を計算
  → Volumetric Fog が有効な場合は IntegratedLightScattering を設定

RenderFog(GraphBuilder, SceneTextures, Views)
  │
  ├─ [A] FFogUniformParameters 生成（View ごと）
  │
  ├─ [B] FHeightFogVS / FHeightFogPS（全画面クワッド）
  │   VS:
  │     FullscreenQuad を描画
  │     → ScreenUV と ViewRay を出力
  │
  │   PS（HeightFog.usf）:
  │     SceneDepth.SampleLevel(ScreenUV) → WorldPos を再構築
  │     RayLength = distance(CameraPos, WorldPos)
  │     if (ApplyVolumetricFog > 0):
  │       FroxelUV = WorldPosToFroxelUV(WorldPos)
  │       (Inscattering, Transmittance) = IntegratedLightScattering.SampleLevel(FroxelUV, 0)
  │     else:
  │       Transmittance = exp(-FogDensity × RayLength × HeightFactor)
  │       Inscattering = InscatteringColor × (1 - Transmittance)
  │       + DirectionalInscattering × DirectionalFactor
  │
  ├─ [C] SceneColor へのブレンド
  │   SceneColor = SceneColor × Transmittance + Inscattering
  │   → Additive ブレンド（深度テストなし）
  │
  └─ [D] SkyAtmosphereAmbientContributionColorScale
      → SkyAtmosphere の色を SceneColor に加算

GetViewFogCommonStartDistance():
  → Volumetric Fog / Local Fog Volume の開始距離を考慮した
    Height Fog の有効開始距離を返す

ShouldRenderFog():
  → r.Fog=1 かつ FogDensity > 0 かつ ヘッドマウントディスプレイ以外
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Fog` | 1 | Height Fog 有効 |
| `r.Fog.DirectionalInscattering` | 1 | 方向散乱有効 |
| `r.Fog.Start` | 0 | Fog 開始距離オーバーライド |

---

## 関連リファレンス

- [[ref_fog_rendering]] — `FFogUniformParameters` / `FHeightFogVS` 詳細
- [[b_volumetric_fog]] — Volumetric Fog Froxel 積分フロー
