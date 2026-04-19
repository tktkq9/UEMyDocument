# SkyAtmosphere ソースマップ

- 対象: 物理ベース大気散乱（Transmittance / MultiScattering / SkyView / AerialPerspective LUT）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[18_skyatmosphere_overview]]

4 段 LUT（Transmittance / MultiScatteredLuminance / SkyView / AerialPerspectiveVolume）を CS で事前計算し、
Sky Pass PS でサンプルして SceneColor に合成。SkyLight キャプチャとも連動する。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/` |
| メイン | `SkyAtmosphereRendering.h/.cpp` |
| Sky Pass | `SkyPassRendering.h/.cpp` |
| SkyLight | `ReflectionEnvironmentCapture.cpp`, `SkyLightRendering.cpp` |
| シェーダー | `Engine/Shaders/Private/SkyAtmosphere.usf` |
| 呼び出し元 | `DeferredShadingRenderer.cpp` |

---

## ファイル → クラス対応

### SkyAtmosphere 本体

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `SkyAtmosphereRendering.h/.cpp` | `RenderSkyAtmosphereMainView()`, `FSkyAtmosphereRenderContext`, `FSkyAtmosphereRenderSceneInfo`, `FAtmosphereUniformShaderParameters` | LUT 構築・Sky 描画オーケストレーション | [[Reference/ref_sky_atmosphere]] |
| `SkyPassRendering.h/.cpp` | `FSkyPassMeshProcessor`, Sky Pass PS | Sky マテリアル描画パス | 同 |

### SkyLight

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `SkyLightRendering.cpp` | `FSkyLightSceneProxy`, `RenderSkyLightDiffuse()` | SkyLight 照明適用 | [[Reference/ref_sky_light]] |
| `ReflectionEnvironmentCapture.cpp` | `ConvolveSpecularSkyLight()` | SkyAtmosphere LUT から SkyLight キューブマップ生成 | 同 |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()
  │
  └─ RenderSkyAtmosphereMainView                SkyAtmosphereRendering.cpp
       │
       ├─[A] TransmittanceLUT CS                256×64 テクスチャ
       │     Beer-Lambert 積分（静的キャッシュ）
       │
       ├─[B] MultiScatteredLuminanceLUT CS      32×32 テクスチャ
       │     多重散乱輝度
       │
       ├─[C] SkyViewLUT CS                       192×108 テクスチャ（毎フレーム）
       │     カメラ視点の空色（Lat/Long）
       │
       ├─[D] AerialPerspectiveVolume CS          32×32×16 フラクセル（毎フレーム）
       │     視線方向 × 深度スライスの散乱・透過
       │
       └─[E] Sky Pass                            SkyPassRendering.cpp
             SkyPassPS → SceneColor の空部分に合成
             bFastSky: SkyViewLUT のみで近似
             bFastAerialPerspective: ボリュームなし近似

[SkyLight キャプチャ]
ReflectionEnvironmentCapture.cpp:ConvolveSpecularSkyLight
  → TransmittanceLUT / SkyViewLUT 参照 → キューブマップ畳み込み
```

---

## 主要 LUT

| LUT | サイズ | 更新頻度 | 対応 CS |
|-----|-------|--------|---------|
| Transmittance | 256×64 | 大気パラメータ変化時 | `SkyAtmosphere.usf:TransmittanceLutCS` |
| MultiScatteredLuminance | 32×32 | ライト方向変化時 | `MultiScatteredLuminanceLutCS` |
| SkyView | 192×108 | 毎フレーム | `SkyViewLutCS` |
| AerialPerspectiveVolume | 32×32×16 | 毎フレーム | `AerialPerspectiveVolumeCS` |

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.SkyAtmosphere` | `SkyAtmosphereRendering.cpp` |
| `r.SkyAtmosphere.FastSkyLUT` | 同 |
| `r.SkyAtmosphere.AerialPerspective.DepthTest` | 同 |
| `r.SkyAtmosphere.SampleCountMin` | 同 |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_sky_atmosphere]] | FSkyAtmosphereRenderContext / FSkyAtmosphereRenderSceneInfo |
| Reference | [[Reference/ref_sky_light]] | FSkyLightSceneProxy / ConvolveSpecularSkyLight |

---

## ue5-dive 起点

- 「大気パラメータ」 → `FAtmosphereUniformShaderParameters`（Rayleigh/Mie/Ozone 係数）
- 「LUT 構築順」 → `SkyAtmosphereRendering.cpp:RenderSkyAtmosphereMainView`
- 「高速モード」 → `r.SkyAtmosphere.FastSkyLUT` + SkyView のみ使用
- 「Sky 描画パス」 → `SkyPassRendering.cpp:FSkyPassMeshProcessor`
- 「SkyLight との連動」 → `ReflectionEnvironmentCapture.cpp:ConvolveSpecularSkyLight`
