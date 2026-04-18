# GPU a: SkyAtmosphere LUTBuild — 大気散乱 LUT 事前計算

- シェーダー: `SkyAtmosphere.usf`（各 CS エントリ）
- CPU 対応: [[a_sky_atmosphere]]
- 上位: [[01_skyatmosphere_gpu_overview]]

---

## 概要

物理ベースの大気散乱（Rayleigh・Mie）を毎フレーム完全計算するのは高コストのため、  
UE5 では複数の **Look Up Table（LUT）** を Compute Shader で事前生成し、  
実際の描画時はそれらをサンプリングするだけで済む構造になっている。

LUT 生成は非同期 Compute で事前実行され、メインの描画パスと並列化される。

---

## LUT の種類と役割

| LUT 名 | フォーマット | 解像度 | 内容 |
|--------|------------|--------|------|
| **TransmittanceLUT** | `R11G11B10F`（PC）/ `RGBA8`（Mobile）| 256×64 | 大気中の光の透過率（高度・仰角の関数）|
| **MultiScatteredLuminanceLUT** | 同上 | 32×32 | 多重散乱の輝度近似テーブル |
| **SkyViewLUT** | 同上 | 192×108 | スカイドームの輝度（仰角・方位角の関数）|
| **DistantSkyLight LUT** | 同上 | 64×64 | Sky Light 用の上半球平均輝度 |
| **AerialPerspective Volume** | `RGBA16F` | 32×32×16 | カメラ距離に応じた光散乱・透過率（3D）|

---

## RenderTransmittanceLutCS（line 1144）

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void RenderTransmittanceLutCS(uint3 ThreadId : SV_DispatchThreadID)
{
    // UV座標 → 高度・仰角への変換（UvToLutTransmittanceParams）
    float2 UV = ThreadId.xy * SkyAtmosphere.TransmittanceLutSizeAndInvSize.zw;
    float ViewHeight, ViewZenithCosAngle;
    UvToLutTransmittanceParams(ViewHeight, ViewZenithCosAngle, UV);

    // 指定方向への光路の光学的深さを積分（IntegrateSingleScatteredLuminance）
    SingleScatteringResult ss = IntegrateSingleScatteredLuminance(...);

    // 透過率 = exp(-光学的深さ)
    float3 Transmittance = exp(-ss.OpticalDepth);
    TransmittanceLutUAV[ThreadId.xy] = Transmittance;
}
```

**使用 CVar:** `r.SkyAtmosphere.TransmittanceLUT.SampleCount`

---

## RenderMultiScatteredLuminanceLutCS（line 1200）

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void RenderMultiScatteredLuminanceLutCS(uint3 ThreadId : SV_DispatchThreadID)
{
    // X軸: ライト仰角のコサイン値
    // Y軸: 大気中の高度（BottomRadius ～ TopRadius）
    float CosLightZenithAngle = UV.x * 2.0f - 1.0f;
    float ViewHeight = Atmosphere.BottomRadiusKm + UV.y * (TopRadius - BottomRadius);

    // 球面上の均等サンプリング（UniformSphereSamplesBuffer）で多重散乱を近似
    // 2次以上の散乱を単純なテーブルで高速近似
}
```

`HIGHQUALITY_MULTISCATTERING_APPROX_ENABLED` が有効な場合はより精密な計算を使用。

---

## RenderSkyViewLutCS（line 1326）

スカイドーム全体の輝度を「仰角 × 方位角」のテーブルとして生成する。  
毎フレームカメラ依存で生成が必要（高度・カメラ位置が影響）。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void RenderSkyViewLutCS(uint3 ThreadId : SV_DispatchThreadID)
{
    // UV → ビュー方向ベクトルへ変換（仰角・方位角）
    float3 WorldDir = UvToSkyViewLutDir(UV);

    // TransmittanceLUT と MultiScattLUT を参照して輝度を計算
    // FASTSKY_ENABLED 時は近似計算
}
```

---

## RenderDistantSkyLightLutCS（line 1404）

Sky Light コンポーネントが大気からキャプチャするための輝度テーブル。  
上半球を分割してサンプリングし、Sky Light の Irradiance を計算する。

---

## RenderCameraAerialPerspectiveVolumeCS（line 1510）

カメラからの距離（奥行き）に応じた大気散乱をボリューム（3D テクスチャ）に格納する。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void RenderCameraAerialPerspectiveVolumeCS(uint3 ThreadId : SV_DispatchThreadID)
{
    // スクリーン位置 + スライス番号 → ワールド空間座標
    // 各スライスでの散乱・透過率を積分
    // 結果を AerialPerspective ボリューム（Texture3DArray）に書き込み
}
```

`FASTAERIALPERSPECTIVE_ENABLED` で高速近似モードに切り替わる。

---

## 非同期 Compute との関係

```
FSkyAtmospherePendingRDGResources（SkyAtmosphereRendering.h:181）
  │
  └─ Nanite ラスタライズ / Pre-pass と並列に LUT 生成を実行
       → RenderSkyAtmosphereLookUpTables() が非同期 Compute キューへ投入
```

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.SkyAtmosphere.FastSky` | 高速スカイ近似（`FASTSKY_ENABLED`）|
| `r.SkyAtmosphere.AerialPerspective.FastApply` | 高速 AerialPerspective（`FASTAERIALPERSPECTIVE_ENABLED`）|
| `r.SkyAtmosphere.TransmittanceLUT.SampleCount` | Transmittance LUT のサンプル数 |
| `r.SkyAtmosphere.MultiScatteringLUT.HighQuality` | 多重散乱 LUT の品質（`HIGHQUALITY_MULTISCATTERING_APPROX_ENABLED`）|
