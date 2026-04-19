# Fog ソースマップ

- 対象: Exponential Height Fog + Volumetric Fog（Froxel 積分）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[19_fog_overview]]

Height Fog は PS で指数関数近似、Volumetric Fog は Froxel（フラスタムボクセル）に散乱・消散を注入 → Z 方向積分 →
`IntegratedLightScattering` Texture3D に出力 → HeightFogPS で統合合成する。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/` |
| Height Fog | `FogRendering.h/.cpp` |
| Volumetric Fog | `VolumetricFog.h/.cpp`, `VolumetricFogVoxelization.cpp` |
| シェーダー | `Engine/Shaders/Private/HeightFogPixelShader.usf`, `VolumetricFog*.usf` |
| 呼び出し元 | `DeferredShadingRenderer.cpp`（Lighting 後） |

---

## ファイル → クラス対応

### Height Fog

| ファイル | 主要関数 / 構造体 | 役割 | 参照 |
|---------|----------------|------|------|
| `FogRendering.h/.cpp` | `SetupFogUniformParameters()`, `RenderFog()`, `FFogUniformParameters`, `FHeightFogVS/PS`, `FExponentialHeightFogSceneInfo` | 指数関数フォグ合成・UB 構築 | [[Reference/ref_fog_rendering]] |

### Volumetric Fog

| ファイル | 主要関数 / 構造体 | 役割 | 参照 |
|---------|----------------|------|------|
| `VolumetricFog.h/.cpp` | `RenderVolumetricFog()`, `FVolumetricFogIntegrationParameters`, `FVolumetricFogGlobalData`, `ComputeZSliceFromDepth()` | Froxel 確保・積分・オーケストレーション | [[Reference/ref_volumetric_fog]] |
| `VolumetricFogVoxelization.cpp` | `VoxelizeVolumeFog CS` | 散乱係数・Emissive を Froxel に注入 | 同 |
| `VolumetricFog.cpp` | `InjectLocalLightShadow CS`, `IntegrateLightScattering CS` | 光源影注入 + Z 方向逐次積分 | 同 |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()（Lighting 完了後）
  │
  ├─[A] SetupFogUniformParameters               FogRendering.cpp
  │
  ├─[B] RenderVolumetricFog                     VolumetricFog.cpp（r.VolumetricFog=1）
  │     │
  │     ├─ Froxel グリッド確保
  │     │    GridSize = (⌈W/8⌉, ⌈H/8⌉, 64)
  │     │    VBufferA: 散乱・消散（RGBA16F）
  │     │    VBufferB: Emissive（RGBA16F）
  │     │
  │     ├─ VoxelizeVolumeFog CS                 VolumetricFogVoxelization.cpp
  │     │    LocalFogVolume / パーティクルの密度注入 + Temporal Jitter
  │     │
  │     ├─ InjectLocalLightShadow CS
  │     │    Shadow Map から影をサンプルして散乱係数をモジュレート
  │     │
  │     └─ IntegrateLightScattering CS
  │          Z 逐次積分 → IntegratedLightScattering Texture3D
  │
  └─[C] RenderFog                               FogRendering.cpp
        FHeightFogPS（全画面）
        → SceneDepth → WorldPos 再構築
        → ApplyVolumetricFog=1 のとき IntegratedLightScattering をサンプル
        → SceneColor に合成
```

---

## Froxel 座標系

```
Froxel = Frustum Voxel
  X,Y:  ViewPort.xy / r.VolumetricFog.GridPixelSize
  Z:    対数分布スライス
    ZSlice = log2(Depth * GridZParams.X + GridZParams.Y) * GridZParams.Z
    （ComputeZSliceFromDepth 関数）
```

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.VolumetricFog` | `VolumetricFog.cpp` |
| `r.VolumetricFog.GridPixelSize` | 同 |
| `r.VolumetricFog.GridSizeZ` | 同 |
| `r.VolumetricFog.Distance` | 同 |
| `r.VolumetricFog.TemporalReprojection` | 同 |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_fog_rendering]] | FFogUniformParameters / FHeightFogVS/PS |
| Reference | [[Reference/ref_volumetric_fog]] | FVolumetricFogIntegrationParameters / FVolumetricFogGlobalData |

---

## ue5-dive 起点

- 「Height Fog の式」 → `HeightFogPixelShader.usf`（`exp(-FogDensity * RayLength * HeightFactor)`）
- 「Volumetric Fog の Froxel 構造」 → `VolumetricFog.cpp:RenderVolumetricFog` + `ComputeZSliceFromDepth`
- 「Z 積分の実装」 → `IntegrateLightScattering CS`（前方向 Transmittance 累積）
- 「LocalFogVolume / パーティクルの密度注入」 → `VolumetricFogVoxelization.cpp:VoxelizeVolumeFog CS`
- 「Height と Volumetric の統合点」 → `FogRendering.cpp:RenderFog` + `ApplyVolumetricFog` フラグ
