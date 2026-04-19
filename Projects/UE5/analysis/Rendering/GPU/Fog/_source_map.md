# GPU Fog ソースマップ

- 対象: Fog GPU シェーダー（ExponentialHeightFog + VolumetricFog Froxel）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_fog_gpu_overview]]

Height Fog は全画面 PS、Volumetric Fog は 3D Froxel（MaterialSetup → LightInjection → LightScattering → FinalIntegration）。
ApplyVolumetricFog=1 時は Height Fog PS が積分済み 3D テクスチャをサンプルして合成する。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/HeightFogVertexShader.usf` |
| シェーダー | `Engine/Shaders/Private/HeightFogPixelShader.usf` |
| シェーダー | `Engine/Shaders/Private/VolumetricFog.usf` |
| シェーダー | `Engine/Shaders/Private/VolumetricFogVoxelization.usf` |
| CPU | `Renderer/Private/FogRendering.cpp` / `VolumetricFog.cpp` |

---

## ファイル → シェーダー対応

### Height Fog

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `HeightFogVertexShader.usf` | `Main()` | `FDeferredShadingSceneRenderer::RenderFog()` | 全画面 VS | [[detail_height_fog]] |
| `HeightFogPixelShader.usf` | `ExponentialPixelMain()` | 同上 | 指数霧計算 + VolumetricFog サンプル合成 | 同 |

### Volumetric Fog

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `VolumetricFog.usf` | `MaterialSetupCS()` | `FDeferredShadingSceneRenderer::RenderVolumetricFog()` | Froxel 密度注入（Height Fog 密度） | [[detail_volumetric_fog]] |
| `VolumetricFog.usf` | `LightScatteringCS()` | 同上 | Directional/Sky/Local ライト散乱計算 | 同 |
| `VolumetricFog.usf` | `FinalIntegrationCS()` | 同上 | カメラ方向レイマーチング積分 | 同 |
| `VolumetricFog.usf` | `InjectShadowedLocalLightPS()` + `WriteToBoundingSphereVS` | 同上 | ローカルライト注入 | 同 |
| `VolumetricFogVoxelization.usf` | 複数 CS | `VoxelizeFogVolumes()` | FogVolume メッシュのボクセル化 | 同 |

---

## GPU データフロー

```
[1] Height Fog（全画面）
    HeightFogVertexShader.usf:Main + HeightFogPixelShader.usf:ExponentialPixelMain
    → SceneDepth → ワールド座標 → 指数霧 → SceneColor 合成

[2] Volumetric Fog
    VolumetricFog.usf:MaterialSetupCS           → VBufferA/B（散乱/消散）
    VolumetricFog.usf:InjectShadowedLocalLightPS → ローカルライト注入
    VolumetricFog.usf:LightScatteringCS         → 散乱値（Directional+Sky+Local）
    VolumetricFog.usf:FinalIntegrationCS        → IntegratedLightScattering (Texture3D)

[3] 合成
    ExponentialPixelMain が ApplyVolumetricFog=1 時に 3D テクスチャをサンプル
```

---

## 主要パーミュテーション

| マクロ | 用途 |
|--------|------|
| `APPLY_VOLUMETRIC_FOG` | Height Fog に Volumetric 結果合成 |
| `SUPPORT_VOLUMETRIC_FOG_SHADOW` | VSM / SM シャドウ対応 |
| `USE_FOG_INSCATTERING_TEXTURE` | Inscattering テクスチャ |
| `USE_EMISSIVE` | エミッシブ参加 |

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_height_fog]] / [[detail_volumetric_fog]] |
| Reference | [[ref_height_fog]] / [[ref_volumetric_fog]] |

---

## ue5-dive 起点

- 「Height Fog 本体」 → `HeightFogPixelShader.usf:ExponentialPixelMain`
- 「Volumetric Fog の散乱」 → `VolumetricFog.usf:LightScatteringCS`
- 「Froxel 積分」 → `VolumetricFog.usf:FinalIntegrationCS`
- 「Local Light 注入」 → `VolumetricFog.usf:InjectShadowedLocalLightPS`
