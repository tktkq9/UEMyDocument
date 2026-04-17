# Sky Light / Reflection Environment シェーダー詳細

- グループ: d - SkyLight
- GPU 概要: [[01_deferred_lighting_gpu_overview]]
- CPU 詳細: [[d_sky_light]]
- リファレンス: [[ref_sky_light]]

---

## 概要

**Sky Light** と **Reflection Environment** は GBuffer から Specular / Diffuse IBL（Image Based Lighting）を計算する。  
Lumen 無効時は ReflectionCapture キューブマップから Specular を取得し、  
Sky Light の SH（球面調和関数）から Diffuse 間接光を適用する。

---

## レンダリングパスの構成

```
Reflection Environment + Sky Light パス
  │
  ├─ [Specular IBL]
  │   PS: ReflectionEnvironmentPixelShader.usf
  │   GBuffer の法線 + Roughness → Reflection Direction → ReflectionCapture.SampleLevel(Mip)
  │   複数 ReflectionCapture をブレンド（半径内にある最大8個）
  │
  ├─ [Sky Light Diffuse（非 Lumen）]
  │   PS: ReflectionEnvironmentPixelShader.usf（Sky Light コードパス）
  │   GBuffer 法線 → SkyLight SH 評価 → Lambert Diffuse 間接光
  │
  └─ [Sky Light Capture（ReflectionCapture 更新）]
      ConvolveSpecularSkyLightCS（ReflectionEnvironmentCapture.usf）
      Sky Atmosphere / Sky Sphere を HDR キューブマップにキャプチャ
      → ミップ生成 → Specular IBL 用にコンボルブ（GGX Importance Sampling）
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `GBufferB` | ShadingModelID / Roughness |
| `GBufferC` | BaseColor（Specular カラー計算用）|
| `ReflectionCapture0〜7` | 各 ReflectionCapture の TextureCube |
| `SkyLightCapture` | Sky Light の HDR キューブマップ |
| `SkyIrradianceSH` | Sky Light の SH 係数（Diffuse 用）|

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| `SceneColor` | `R11G11B10F` | Specular IBL + Sky Light Diffuse |

---

## シェーダーコアロジック

### Specular IBL 評価

```hlsl
// GBuffer の法線と Roughness から反射ベクトル計算
float3 ReflectVector = reflect(-CameraVector, GBuffer.WorldNormal);

// SpecularOcclusion（AOを考慮したスペキュラー遮蔽）
float SpecularOcclusion = GetSpecularOcclusion(GBuffer.WorldNormal, CameraVector, GBuffer.Roughness);

// Roughness → ミップレベル変換
float MipLevel = ComputeReflectionCaptureMipFromRoughness(GBuffer.Roughness);

// ReflectionCapture サンプリング（最大8個をブレンド）
float3 SpecularIBL = 0;
for (int i = 0; i < NumCaptures; i++)
{
    float3 CaptureColor = ReflectionCaptures[i].SampleLevel(ReflectVector, MipLevel).rgb;
    float CaptureWeight = GetCaptureWeight(WorldPosition, CaptureData[i]);
    SpecularIBL += CaptureColor * CaptureWeight;
}

// Pre-Integrated GF テクスチャ（Fresnel + Visibility の近似）
float2 GF = PreIntegratedGF.SampleLevel(float2(NoV, Roughness), 0);
SpecularIBL *= GBuffer.F0 * GF.x + GF.y;
```

### Sky Light Diffuse（SH 評価）

```hlsl
// SkyIrradianceSH から Diffuse 間接光を計算
float3 SkyIrradiance = GetSkyIrradiance(GBuffer.WorldNormal, SkyIrradianceSH);
float3 SkyDiffuse = SkyIrradiance * GBuffer.DiffuseColor;
```

---

## CPU 呼び出しの流れ

```
RenderReflections()                          // ReflectionEnvironment.cpp
  │
  ├─ [ReflectionCapture キャプチャ更新]
  │   UpdateReflectionCaptureContents()
  │   → ConvolveSpecularSkyLightCS() で GGX コンボリューション
  │
  └─ AddPass(RDG_EVENT_NAME("ReflectionEnvironment"))
      PS: ReflectionEnvironmentPixelShader.usf
      Specular IBL + Sky Light Diffuse を SceneColor に加算
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `ReflectionEnvironmentPixelShader.usf` | PS | Specular IBL + Sky Light Diffuse |
| `ReflectionEnvironmentShaders.usf` | CS | ReflectionCapture コンボリューション |
| `ReflectionEnvironmentShared.ush` | ヘッダ | キャプチャウェイト・Mip レベル計算 |
