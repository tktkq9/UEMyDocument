# a: Sky Atmosphere 描画（RenderSkyAtmosphere）

- 対象ファイル: `SkyAtmosphereRendering.h/.cpp`
- 概要: [[18_skyatmosphere_overview]]

---

## 概要

`RenderSkyAtmosphereMainView()` は Transmittance / MultiScattered / SkyView / AerialPerspective の  
4 種の LUT をパラメータの変化に応じてキャッシュしながら更新し、  
最終的に RenderSky() で SceneColor に合成する。

---

## FSkyAtmosphereRenderContext（SkyAtmosphereRendering.h）

```cpp
struct FSkyAtmosphereRenderContext
{
    // ─── 描画制御フラグ ──────────────────────────────────────────
    bool bUseDepthBoundTestIfPossible;         // Depth Bounds Test で空部分のみ描画
    bool bForceRayMarching;                    // LUT をバイパスして Ray March
    bool bDepthReadDisabled;                   // SceneDepth を参照しない（キャプチャ用）
    bool bDisableBlending;                     // ブレンドなし（Sky Capture 用）
    bool bFastSky;                             // SkyViewLUT のみ（近似高速版）
    bool bFastAerialPerspective;               // AerialPerspective ボリュームを使わない
    bool bFastAerialPerspectiveDepthTest;
    bool bSecondAtmosphereLightEnabled;        // 第二光源（月など）有効
    bool bShouldSampleOpaqueShadow;            // 不透明シャドウをサンプル

    // ─── LUT テクスチャ参照 ──────────────────────────────────────
    FRDGTextureRef TransmittanceLut;
    FRDGTextureRef MultiScatteredLuminanceLut;
    FRDGTextureRef SkyAtmosphereViewLutTexture;           // SkyView LUT
    FRDGTextureRef SkyAtmosphereCameraAerialPerspectiveVolume;     // 全散乱
    FRDGTextureRef SkyAtmosphereCameraAerialPerspectiveVolumeMieOnly;  // Mie のみ
    FRDGTextureRef SkyAtmosphereCameraAerialPerspectiveVolumeRayOnly;  // Rayleigh のみ

    // ─── ビューパラメータ ────────────────────────────────────────
    FViewMatrices* ViewMatrices;
    TUniformBufferRef<FViewUniformShaderParameters> ViewUniformBuffer;
    bool bSceneHasSkyMaterial;
};
```

---

## 4 種の LUT 生成フロー

### [A] TransmittanceLUT（静的キャッシュ）

```
生成条件: 大気パラメータ変化時のみ

CS: RenderTransmittanceLutCS.usf
解像度: 256×64
各テクセル (u, v):
  u → 大気底から視点の天頂角
  v → 大気底からの高度
  T(x→top) = exp(-∫₀^h σ_ext(s) ds)
  σ_ext = Rayleigh + Mie 消散係数の密度重み付き積分

出力: R16G16B16A16F（RGB = Transmittance, A = 未使用）
```

### [B] MultiScatteredLuminanceLUT

```
生成条件: 大気パラメータ or 太陽方向が変化した場合

CS: RenderMultiScatteredLuminanceLutCS.usf
解像度: 32×32
各テクセル: 多重散乱による輝度（無限回のバウンス和）
手法: 逐次近似（Iteration × MultipliedBy係数）

入力: TransmittanceLUT
出力: R11G11B10F（RGB = 多重散乱輝度）
```

### [C] SkyViewLUT（毎フレーム）

```
CS: RenderSkyViewLutCS.usf
解像度: 192×108（またはプラットフォーム依存）
座標系: 等積度射影（緯経度マップ）
  u → 方位角
  v → 仰角

各テクセル = カメラ位置からのその方向の空の色:
  = 単一散乱 + 多重散乱（MultiScatteredLuminanceLUT からサンプル）
  + Transmittance × 太陽ディスク輝度

入力: TransmittanceLUT, MultiScatteredLuminanceLUT
出力: R11G11B10F（空の色、対数スケール）
```

### [D] AerialPerspectiveVolume（毎フレーム）

```
CS: RenderCameraAerialPerspectiveVolumeCS.usf
解像度: 32×32×16 フラクセル
深度分布: 等対数分布（近距離は細かく、遠距離は粗く）

各フラクセル = カメラから各深度スライスまでの:
  - 累積散乱色（Inscattering）
  - 累積透過率（Transmittance）

入力: TransmittanceLUT, MultiScatteredLuminanceLUT
出力: R16G16B16A16F（RGB = Inscattering, A = Transmittance）
```

---

## RenderSky() フロー（描画パス）

```
RenderSky(GraphBuilder, View, RenderContext, SceneColorTexture)
  │
  ├─ [A] DepthBoundsTest 設定（bUseDepthBoundTestIfPossible）
  │   → スカイドームより前の深度を持つピクセルをスキップ
  │
  ├─ [B] FSkyAtmospherePS でフルスクリーン描画
  │   [bFastSky=false（高品質）]:
  │     SkyViewLUT.SampleLevel(LightDirection UV) + AerialPerspective
  │   [bFastSky=true（高速近似）]:
  │     SkyViewLUT のみ（Aerial Perspective をスキップ）
  │   [bForceRayMarching=true]:
  │     LUT を使わずフル Ray March（最高品質）
  │
  └─ [C] SceneColor にアルファブレンドで合成
      Sky が見える部分（SceneDepth = FarPlane）のみ書き込み
```

---

## 関連リファレンス

- [[ref_sky_atmosphere]] — `FAtmosphereUniformShaderParameters` / `FSkyAtmosphereRenderSceneInfo`
- [[b_sky_light]] — SkyLight キャプチャ・ConvolveSpecular
