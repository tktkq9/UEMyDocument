# Volumetric Fog シェーダー詳細

- グループ: b - Volumetric Fog
- GPU 概要: [[01_fog_gpu_overview]]
- CPU 詳細: [[b_volumetric_fog]]
- リファレンス: [[ref_volumetric_fog]]

---

## 概要

3D グリッド（Froxel = Frustum-aligned Voxel）に散乱・消散値を注入し、  
カメラ方向にレイマーチングで積分して `IntegratedLightScattering`（Texture3D）を生成する。  
HeightFog パスで SceneDepth に対応する Froxel UV をサンプルして最終合成される。

---

## レンダリングパスの構成

```
Volumetric Fog パイプライン
  │
  ├─ [1] Material Setup CS（Froxel 密度初期化）
  │   CS: MaterialSetupCS（VolumetricFog.usf:123）
  │   ExponentialHeightFog の密度 → VBufferA.a（消散係数）
  │   Albedo / Emissive → VBufferA.rgb / VBufferB.rgb
  │
  ├─ [2] LocalFogVolume Voxelization（オプション）
  │   CS: VolumetricFogVoxelization.usf（各 LocalFogVolume の形状で密度注入）
  │
  ├─ [3] Local Light Injection
  │   VS: WriteToBoundingSphereVS（VolumetricFog.usf:278）
  │   PS: InjectShadowedLocalLightPS（VolumetricFog.usf:542）
  │   ローカルライト（Point/Spot/Rect）の散乱を Froxel に注入
  │
  ├─ [4] Light Scattering CS（メインの光散乱計算）
  │   CS: LightScatteringCS（VolumetricFog.usf:767）
  │   ディレクショナルライト + スカイライト + ローカルライト GridLight の散乱値を積分
  │   Temporal Reprojection（前フレームとブレンド）も実施
  │
  └─ [5] Final Integration CS（レイマーチング積分）
      CS: FinalIntegrationCS（VolumetricFog.usf:1096）
      Froxel を手前 → 奥の順に積分して透過率・散乱色を蓄積
      出力: IntegratedLightScattering（Texture3D）
```

---

## 入出力

### Froxel バッファ

| バッファ | フォーマット | 内容 |
|---------|------------|------|
| `VBufferA` | `RWTexture3D<float4>` | rgb = 散乱係数 σ_s, a = 消散係数 σ_e |
| `VBufferB` | `RWTexture3D<float4>` | rgb = Emissive, a = Henyey-Greenstein g |
| `LightScattering` | `RWTexture3D<float4>` | rgb = 散乱照度, a = 透過率（積分途中）|

### 最終出力

| バッファ | フォーマット | 内容 |
|---------|------------|------|
| `IntegratedLightScattering` | `Texture3D<float4>` | rgb = 積分済み散乱色, a = 積分済み透過率 |

---

## シェーダーコアロジック

### [1] MaterialSetupCS（:123）

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, THREADGROUP_SIZE)]
void MaterialSetupCS(uint3 GridCoordinate : SV_DispatchThreadID)
{
    // Froxel のワールド座標を計算
    float SceneDepth;
    float3 WorldPos = ComputeCellTranslatedWorldPosition(GridCoordinate, 0.5, SceneDepth);

    // ExponentialHeightFog の密度を Froxel に注入
    float Extinction = FogDensity * exp(-FogHeightFalloff * (WorldPos.z - FogHeight));
    float3 Albedo = FogScatteringAlbedo;

    RWVBufferA[GridCoordinate] = float4(Albedo * Extinction, Extinction);
    RWVBufferB[GridCoordinate] = float4(FogEmissive, FogPhaseG);
}
```

### [4] LightScatteringCS（:767）

```hlsl
[numthreads(THREADGROUP_SIZE_X, THREADGROUP_SIZE_Y, THREADGROUP_SIZE_Z)]
void LightScatteringCS(uint3 GridCoordinate : SV_DispatchThreadID)
{
    float3 WorldPos = ComputeCellTranslatedWorldPosition(GridCoordinate, JitteredOffset);
    float4 VBufferA_Val = VBufferA[GridCoordinate];  // 散乱・消散
    float4 VBufferB_Val = VBufferB[GridCoordinate];  // Emissive・PhaseG

    // ディレクショナルライト散乱
    float3 DirectionalScattering = ComputeDirectionalLightScattering(WorldPos, VBufferA_Val, ...);

    // ローカルライト散乱（Light Grid 経由）
    float3 LocalScattering = ComputeLocalLightScattering(WorldPos, GridCoordinate, VBufferA_Val, VBufferB_Val);

    // Emissive + Temporal Reprojection
    float3 TotalScattering = DirectionalScattering + LocalScattering + VBufferB_Val.rgb;
    float3 HistoryScattering = SampleHistoryLightScattering(WorldPos, GridCoordinate);
    TotalScattering = lerp(TotalScattering, HistoryScattering, HistoryWeight);

    RWLightScattering[GridCoordinate] = float4(TotalScattering, VBufferA_Val.a);
}
```

### [5] FinalIntegrationCS（:1096）

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void FinalIntegrationCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    uint2 XY = DispatchThreadId.xy;
    float AccumulatedTransmittance = 1.0;
    float3 AccumulatedScattering = 0;

    // Z スライスを手前から奥へ積分（レイマーチング）
    for (uint Z = 0; Z < GridSizeZ; Z++)
    {
        uint3 GridCoord = uint3(XY, Z);
        float4 Scattering = LightScattering[GridCoord];

        // Beer-Lambert: T = exp(-sigma_e * dz)
        float SliceDepth = ComputeDepthFromZSlice(Z);
        float dz = SliceDepth - PrevSliceDepth;
        float SliceTransmittance = exp(-Scattering.a * dz);

        // 散乱の寄与 = 散乱色 × 区間内の透過率積分
        AccumulatedScattering += Scattering.rgb * AccumulatedTransmittance * (1 - SliceTransmittance) / max(Scattering.a, 0.00001);
        AccumulatedTransmittance *= SliceTransmittance;

        RWIntegratedLightScattering[GridCoord] = float4(AccumulatedScattering, AccumulatedTransmittance);
    }
}
```

---

## CPU 呼び出しの流れ

```
RenderVolumetricFog()                        // VolumetricFogRendering.cpp
  │
  ├─ AllocateFroxelBuffers(VBufferA, VBufferB, LightScattering, IntegratedLightScattering)
  │
  ├─ AddPass(MaterialSetupCS)       → VBufferA/B 初期化
  ├─ VoxelizeFogVolumes()           → LocalFogVolume 密度追加
  ├─ AddPass(InjectShadowedLocalLightPS) × ローカルライト数
  ├─ AddPass(LightScatteringCS)     → 光散乱計算
  └─ AddPass(FinalIntegrationCS)    → 積分
       → IntegratedLightScattering を FFogUniformParameters にバインド
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `VolumetricFog.usf` | CS/PS/VS | Froxel 生成・光散乱・積分 本体 |
| `VolumetricFogVoxelization.usf` | CS | LocalFogVolume の形状を Froxel に注入 |
| `VolumetricFogLightFunction.usf` | CS | Light Function マテリアルの適用 |
| `HeightFogCommon.ush` | ヘッダ | Height Fog 密度計算（MaterialSetupCS で使用）|
| `LightGridCommon.ush` | ヘッダ | ローカルライトの Light Grid アクセス |
