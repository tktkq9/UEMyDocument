# GPU c: SkyLight — Sky Light キャプチャ・ConvolveSkyLight

- シェーダー: `SkyAtmosphere.usf`（`RenderDistantSkyLightLutCS`）, `ReflectionEnvironmentShaders.usf`, `DistanceFieldAmbientOcclusion.usf`
- CPU 対応: [[b_sky_light]]
- 上位: [[01_skyatmosphere_gpu_overview]]

---

## 概要

Sky Light は「大気から来る環境光」を SkyAtmosphere から取得してシーン全体に適用する。  
GPU パスは以下の2段階からなる：
1. `RenderDistantSkyLightLutCS` — 大気輝度テーブルを生成
2. ConvolveSkyLight 系 — キューブマップへの変換・畳み込みで Sky Light Irradiance を生成

---

## RenderDistantSkyLightLutCS（line 1404）

大気の「遠方から来る光」（直接光・散乱光）を LUT に格納する。  
Sky Light コンポーネントが大気をキャプチャする際に参照される。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void RenderDistantSkyLightLutCS(uint3 ThreadId : SV_DispatchThreadID)
{
    float2 UV = ThreadId.xy * LutSizeAndInvSize.zw;

    // UV → 上半球の方向ベクトルへ変換
    float3 WorldDir = UvToSkyViewLutDir(UV);

    // TransmittanceLUT + MultiScattLUT を参照して
    // 指定方向から来る大気輝度を計算
    float3 Luminance = ComputeAtmosphereLuminance(WorldDir, ...);

    DistantSkyLightLutUAV[ThreadId.xy] = float4(Luminance, 0);
}
```

---

## Sky Light キューブマップキャプチャフロー

```
RenderDistantSkyLightLutCS
  │ → DistantSkyLight LUT (Texture2D)
  │
  ↓ CaptureOrUpdateSkyOnce()
CaptureIrradianceFromAtmosphere()
  │ → LUT を 6 面キューブマップへ再投影
  │
  ↓
FilterReflectionEnvironment()  （ReflectionEnvironmentShaders.usf）
  → スペキュラ用にキューブマップを Mip フィルタリング
     → ReflectionCapture として Skylight Irradiance に使用
```

---

## ConvolveSkyLight（ReflectionEnvironmentShaders.usf）

Sky Light のキューブマップを Diffuse Irradiance（球面調和関数）と  
Specular（Mip 階層）に変換する。

```hlsl
// スペキュラ用キューブマップ Mip 生成
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ConvolveSpecularSkyLightCS(uint3 ThreadId : SV_DispatchThreadID)
{
    // GGX BRDF でフィルタリング（Roughness = sqrt(Mip / MaxMip)）
    float Roughness = sqrt((float)MipIndex / (float)MaxMipCount);
    // 重要サンプリングで入力キューブマップを積分
}
```

---

## Sky AO（Distance Field AO との接続）

Sky Light が有効かつ DFAO が有効な場合、  
DFAO の BentNormal テクスチャを Sky Light の Irradiance に適用して AO を実現する。

```
ShouldRenderDistanceFieldAO() → true
  │
  └─ RenderDistanceFieldLighting()
       → BentNormalAO テクスチャを生成
          → RenderDeferredReflectionsAndSkyLighting() で
             SkyLight Irradiance × BentNormal → 最終 AO
```

---

## Sky Light 更新タイミング

| 条件 | 更新 |
|------|------|
| `bRealTimeCapture = true` | 毎フレーム大気から再キャプチャ |
| `bRealTimeCapture = false` | キャプチャ時のみ（静的）|
| Sky 変数（ライト角度等）変化 | 次フレームで再生成トリガー |

---

## SkyAtmosphere × SkyLight の関係

```
USkyAtmosphereComponent
  → 大気物理パラメータを提供（散乱係数・大気厚さ等）

USkyLightComponent（bRealTimeCapture = true 時）
  → SkyAtmosphere の LUT を参照して自動的に Irradiance を更新
  → Lumen / DF AO / Reflection Environment に光源データを供給
```
