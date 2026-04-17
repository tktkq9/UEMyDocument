# b: Sky Light キャプチャ・ConvolveSpecularSkyLight

- 対象ファイル: `ReflectionEnvironmentCapture.h/.cpp` / `SkyAtmosphereRendering.cpp`
- 概要: [[18_skyatmosphere_overview]]

---

## 概要

**Sky Light** は空の輝度をキャプチャしてシーンの間接光に使用する。  
`FSkyLightSceneProxy` が定期的にキューブマップキャプチャを行い、  
`ConvolveSpecularSkyLight()` で各 Roughness 値に対応した Mip フィルタリングを行う。

---

## FSkyLightSceneProxy の役割

```cpp
// SkyLight.h（Engine 側）
// FSkyLightSceneProxy はレンダースレッドでの SkyLight の表現

class FSkyLightSceneProxy
{
    // ─── キャプチャ設定 ──────────────────────────────────────────
    float BlendFraction;                  // リアルタイムキャプチャのブレンド割合
    bool bRealTimeCaptureEnabled;         // リアルタイムキャプチャ有効
    bool bCastShadows;                    // シャドウキャスト
    bool bCastRaytracedShadow;            // Ray Traced Shadow

    // ─── キューブマップ ──────────────────────────────────────────
    FTexture* ProcessedSkyTexture;        // フィルタ済みキューブマップ（事前焼き込み）
    FTextureCubeRHIRef ProcessedSkyTextureRef; // RHI テクスチャ
    FSHVectorRGB3 IrradianceEnvironmentMap; // SH 係数（間接拡散光）
    float AverageBrightness;

    // ─── Sky Atmosphere 連携 ─────────────────────────────────────
    bool bHasStaticSkyLight;              // 静的 SkyLight か
    FSkyAtmosphereRenderSceneInfo* SkyAtmosphereInfo; // 大気情報へのポインタ
};
```

---

## SkyLight キャプチャフロー

```
[変化検知]
FSkyLightSceneProxy::SetCaptureDirty()
  → bHasBeenRecaptured = false でマーク
  → 次のフレームで再キャプチャをトリガー

FScene::UpdateSkyCaptureContents()      SceneRendering.cpp
  → bHasBeenRecaptured=false の SkyLight を収集
  → CaptureSceneIntoScratchCubemap() で 6 面キャプチャ

[6面キャプチャ]
CaptureSceneIntoScratchCubemap()        ReflectionEnvironmentCapture.cpp
  → for each CubeFace in 6:
    → CaptureFaceView でその方向のシーンを描画
    → Sky Atmosphere LUT を参照して空を合成
    → ScratchCubemapTexture に格納

[Convolve（MipFilter）]
ConvolveSpecularSkyLight(ScratchCubemapTexture)
  → for each Mip in [0, MipCount):
    → Roughness = Mip / MipCount
    → GGX 分布でキューブマップをフィルタ（Importance Sampling）
    → 低 Roughness = 鏡面反射, 高 Roughness = 拡散反射に近づく
  → ProcessedSkyTexture に書き込み

[SH 係数計算]
ComputeSkyEnvMapDiffuseSH()
  → IrradianceEnvironmentMap（L2 SH係数）を計算
  → DeferredLighting でランバート拡散光として利用
```

---

## SkyLight の利用（DeferredLighting）

```
[スペキュラ反射]
ReflectionEnvironment.usf:
  float Roughness = GBufferB.a
  int MipLevel = Roughness × (SkyLightCubemapMips - 1)
  float3 Reflection = SkyLightCubemap.SampleLevel(ReflectDir, MipLevel)
  → PreIntegratedGFTexture（GGX E(μ,ρ)）でエネルギー保存

[拡散間接光]
BasePassPixelShader.usf:
  float3 Irradiance = SHEval(IrradianceEnvironmentMap, Normal)
  → Lambertian 拡散光として IndirectIrradiance に加算

[遮蔽]
  SkyOcclusionColor（DFAO / Lumen AO で計算）を乗算
  → r.DFAO.SpecularOcclusionMode で制御
```

---

## Sky Atmosphere との連携

```
SkyLight がリアルタイムキャプチャ（bRealTimeCaptureEnabled）の場合:
  → キャプチャ時に SkyAtmosphere の LUT を使って空の色を正確に計算
  → TransmittanceLUT / SkyViewLUT を参照（bForRealtimeSkyCapture=true）

静的 SkyLight の場合:
  → 事前にキャプチャして HDR テクスチャとして保存
  → ランタイムは ProcessedSkyTexture を参照するだけ
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.SkyLight.RealTimeReflectionCapture` | 0 | リアルタイムキャプチャ有効 |
| `r.SkyLight.CubemapResolution` | 128 | キャプチャキューブマップ解像度 |
| `r.DFAO.SpecularOcclusionMode` | 1 | スペキュラ AO モード |
| `r.SkyLightingQuality` | 1 | Sky Lighting 品質（0=低速, 1=通常）|

---

## 関連リファレンス

- [[ref_sky_light]] — `FSkyLightParameters` / `FReflectionUniformParameters`
- [[a_sky_atmosphere]] — SkyAtmosphere LUT 生成・RenderSky() フロー
