# Height Fog シェーダー詳細

- グループ: a - Height Fog
- GPU 概要: [[01_fog_gpu_overview]]
- CPU 詳細: [[a_height_fog]]
- リファレンス: [[ref_height_fog]]

---

## 概要

**ExponentialHeightFog** の GPU 実装。フルスクリーンクワッドを描画して  
各ピクセルの SceneDepth から深度を読み、指数関数的に霧密度を計算して SceneColor に合成する。  
Volumetric Fog が有効な場合は `IntegratedLightScattering` Texture3D をサンプルして結果を上書きする。

---

## レンダリングパスの構成

```
Height Fog フルスクリーンパス
  │
  ├─ VS: HeightFogVertexShader.usf#Main()
  │   フルスクリーントライアングル → TexCoord, ScreenVector を出力
  │
  └─ PS: HeightFogPixelShader.usf#ExponentialPixelMain()
      ├─ SceneDepth を読み取り
      ├─ ScreenVector → ワールドレイ方向を計算
      ├─ [ApplyVolumetricFog == 0] 指数霧を計算
      │   ExponentialFog(RayLength, FogHeight, FogDensity, HeightFalloff)
      └─ [ApplyVolumetricFog == 1] Froxel UV を計算して LightScattering3D をサンプル
          FroxelUV = TranslatedWorldToFroxelUV(WorldPos)
          (Inscattering, Transmittance) = IntegratedLightScattering.SampleLevel(FroxelUV, 0)
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `SceneDepthTexture` | シーン深度（SceneTextures から）|
| `FFogUniformParameters` | 指数霧パラメータ（密度・高さ・色等）|
| `IntegratedLightScattering` | Volumetric Fog の積分済みボリューム（Texture3D）|
| `OcclusionTexture` | AO や空オクルージョンテクスチャ（オプション）|

### 出力

| リソース | 説明 |
|---------|------|
| `OutColor` | 霧色 × (1-T) + SceneColor × T（T = 透過率）|

---

## シェーダーコアロジック（HeightFogPixelShader.usf）

### ExponentialPixelMain

```hlsl
void ExponentialPixelMain(
    float2 TexCoord    : TEXCOORD0,
    float3 ScreenVector: TEXCOORD1,
    float4 SVPos       : SV_POSITION,
    out float4 OutColor: SV_Target0)
{
    // 1. SceneDepth 読み取り
    float DeviceZ = Texture2DSampleLevel(SceneDepthTexture, TexCoord, 0).r;
    float SceneDepth = ConvertFromDeviceZ(DeviceZ);

    // 2. ワールド位置の再構成
    float3 WorldRay = ScreenVector * SceneDepth;
    float3 WorldPos = View.TranslatedWorldCameraOrigin + WorldRay;

    // 3. 指数霧 or Volumetric Fog
    float4 FogColor;
    if (FogUniforms.ApplyVolumetricFog > 0)
    {
        // Froxel サンプリング
        float3 FroxelUV = TranslatedWorldToVolumetricFogUV(WorldPos);
        float4 Scattering = IntegratedLightScattering.SampleLevel(Sampler, FroxelUV, 0);
        // Scattering.rgb = 散乱色, Scattering.a = 透過率
        FogColor = float4(Scattering.rgb, 1.0 - Scattering.a);
    }
    else
    {
        // 指数霧計算
        float2 ExponentialFog = GetExponentialHeightFog(WorldPos, ...);
        // ExponentialFog.x = 透過率T, ExponentialFog.y = 方向散乱係数
        FogColor = float4(FogUniforms.InscatteringColor * (1 - ExponentialFog.x), 1 - ExponentialFog.x);
    }

    OutColor = FogColor;
}
```

### 指数霧計算（HeightFogCommon.ush）

```hlsl
// 高さによる密度の指数的減衰
// FogDensity × exp(-FogHeightFalloff × (RayStart.z - FogHeight))
// レイ長さ × 密度 の積分値が透過率 T を決める
float2 GetExponentialHeightFog(float3 WorldPos, float3 CameraPos, FFogUniformParameters Fog)
{
    float FogRayLength = length(WorldPos - CameraPos);
    float StartHeight = CameraPos.z - Fog.FogHeight;
    float Exponent = max(-127.0f, Fog.FogHeightFalloff * StartHeight);
    float LineIntegral = Fog.FogDensity * exp2(-Exponent) * ...;
    float T = exp2(-LineIntegral);  // 透過率
    return float2(T, DirectionalInscatteringCoeff);
}
```

---

## CPU 呼び出しの流れ

```
RenderFog()                                  // FogRendering.cpp
  │
  ├─ ShouldRenderFog() → false なら early out
  ├─ CreateFogUniformBuffer() → FFogUniformParameters バインド
  │   ApplyVolumetricFog = bShouldRenderVolumetricFog ? 1.0f : 0.0f
  │
  ├─ AddPass(RDG_EVENT_NAME("HeightFog"))
  │   VS: HeightFogVertexShader.usf#Main()
  │   PS: HeightFogPixelShader.usf#ExponentialPixelMain()
  │   RT: SceneColor（アディティブブレンド）
  │
  └─ [Volumetric Fog 有効時]
      PassParameters->IntegratedLightScattering = VolumetricFogResult
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `HeightFogPixelShader.usf` | PS | 指数霧 / Volumetric Fog サンプリング |
| `HeightFogVertexShader.usf` | VS | フルスクリーンクワッドの UV・ViewRay 出力 |
| `HeightFogCommon.ush` | ヘッダ | 指数霧の密度計算・高さ積分ユーティリティ |
| `VolumetricFog.usf` | CS/PS | Volumetric Fog のボリューム生成（別パス）|
