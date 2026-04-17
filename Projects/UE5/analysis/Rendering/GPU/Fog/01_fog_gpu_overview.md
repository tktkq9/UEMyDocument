# Fog GPU 処理概要

- グループ: Fog GPU
- CPU 概要: [[19_fog_overview]]
- CPU 詳細: [[a_height_fog]] | [[b_volumetric_fog]]

---

## Fog GPU パイプライン実行順

UE5 の Fog は **ExponentialHeightFog（全画面フルスクリーン合成）** と  
**VolumetricFog（Froxel ベースの3D ボリュームライティング）** の2系統がある。

```
[Height Fog]
[1]  Exponential Height Fog（全画面パス）
     └─ ExponentialPixelMain（HeightFogPixelShader.usf）
          SceneDepth → ワールド座標復元 → 指数霧計算 → SceneColor 合成

[Volumetric Fog]
[2]  VBuffer Material Setup（Froxel 密度注入）
     └─ MaterialSetupCS（VolumetricFog.usf）
          ExponentialHeightFog 密度 → VBufferA（散乱・消散）

[3]  Light Injection（光の散乱注入）
     ├─ WriteToBoundingSphereVS + InjectShadowedLocalLightPS
     │   ローカルライト（Point/Spot）を Froxel ボリュームに注入
     └─ LightScatteringCS（VolumetricFog.usf）
          ディレクショナルライト・スカイライト・ローカルライト → 散乱値を計算

[4]  Final Integration（レイマーチング積分）
     └─ FinalIntegrationCS（VolumetricFog.usf）
          Froxel をカメラ方向に積分 → 積分済み散乱テクスチャ（LightScattering 3D）

[合成]
[5]  Fog Apply（Height Fog + Volumetric Fog 結果合成）
     └─ ExponentialPixelMain が ApplyVolumetricFog=1 時に LightScattering3D をサンプル
```

---

## 各ステップの詳細

### [1] Exponential Height Fog

| 項目 | 内容 |
|-----|------|
| **概要** | 全画面フルスクリーンパス。SceneDepth からワールド座標を復元して指数霧計算 |
| **CPU 関数** | `FDeferredShadingSceneRenderer::RenderFog()` |
| **シェーダー** | VS: `HeightFogVertexShader.usf#Main()`; PS: `HeightFogPixelShader.usf#ExponentialPixelMain()` |
| **出力** | `SceneColor`（霧の色と透過率を合算）|
| **CPU 詳細** | [[a_height_fog]] |
| **GPU シェーダー詳細** | [[detail_height_fog]] / [[ref_height_fog]] |

---

### [2〜4] Volumetric Fog

| 項目 | 内容 |
|-----|------|
| **概要** | 3D Froxel グリッドに散乱・消散値を注入し、レイマーチングで積分 |
| **CPU 関数** | `FDeferredShadingSceneRenderer::RenderVolumetricFog()` |
| **シェーダー** | `VolumetricFog.usf`: `MaterialSetupCS` / `LightScatteringCS` / `FinalIntegrationCS` |
| **出力** | `IntegratedLightScattering`（Texture3D: 積分済み散乱・透過率）|
| **CPU 詳細** | [[b_volumetric_fog]] |
| **GPU シェーダー詳細** | [[detail_volumetric_fog]] / [[ref_volumetric_fog]] |

---

## シェーダー別 CPU 対応一覧

| シェーダーファイル | 主要エントリポイント | CPU 関数 | グループ |
|----------------|----------------|---------|---------|
| `HeightFogVertexShader.usf` | `Main()` | `RenderFog()` | [[a_HeightFog]] |
| `HeightFogPixelShader.usf` | `ExponentialPixelMain()` | `RenderFog()` | [[a_HeightFog]] |
| `VolumetricFog.usf` | `MaterialSetupCS()` | `RenderVolumetricFog()` | [[b_VolumetricFog]] |
| `VolumetricFog.usf` | `LightScatteringCS()` | `RenderVolumetricFog()` | [[b_VolumetricFog]] |
| `VolumetricFog.usf` | `FinalIntegrationCS()` | `RenderVolumetricFog()` | [[b_VolumetricFog]] |
| `VolumetricFog.usf` | `InjectShadowedLocalLightPS()` | `RenderVolumetricFog()` | [[b_VolumetricFog]] |
| `VolumetricFogVoxelization.usf` | 複数 CS | `VoxelizeFogVolumes()` | [[b_VolumetricFog]] |
