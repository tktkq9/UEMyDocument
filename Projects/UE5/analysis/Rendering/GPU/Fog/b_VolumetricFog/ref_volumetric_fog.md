# REF: Volumetric Fog シェーダー

- グループ: b - Volumetric Fog
- 詳細: [[detail_volumetric_fog]]
- CPU リファレンス: [[ref_volumetric_fog]]
- ソース: `Engine/Shaders/Private/VolumetricFog.usf`  
          `Engine/Shaders/Private/VolumetricFogVoxelization.usf`

---

## MaterialSetupCS（VolumetricFog.usf:123）

### エントリポイント

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, THREADGROUP_SIZE)]
void MaterialSetupCS(uint3 GridCoordinate : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Compute Shader |
| **スレッドグループ** | 4×4×4 = 64 threads |
| **目的** | ExponentialHeightFog の密度パラメータを Froxel VBufferA/B に初期化 |

#### 出力バインド

| バインド名 | 型 | 説明 |
|-----------|------|------|
| `RWVBufferA` | `RWTexture3D<float4>` | rgb=散乱係数, a=消散係数 |
| `RWVBufferB` | `RWTexture3D<float4>` | rgb=Emissive, a=PhaseG（USE_EMISSIVE 時のみ）|

---

## WriteToBoundingSphereVS（:278）

```hlsl
void WriteToBoundingSphereVS(
    float4 InPosition : ATTRIBUTE0,
    uint InstanceId   : SV_InstanceID,
    out float4 OutPosition : SV_POSITION,
    ...)
```

| 項目 | 内容 |
|-----|------|
| **目的** | ローカルライトのバウンディングスフィアを Froxel グリッドに投影 |
| **Instance** | ローカルライト 1 つ = 1 インスタンス |

---

## InjectShadowedLocalLightPS（:542）

```hlsl
void InjectShadowedLocalLightPS(
    float4 SvPosition : SV_POSITION,
    float  LayerIndex : SV_RenderTargetArrayIndex,
    out float4 OutScattering : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Pixel Shader（Froxel スライスにバインド）|
| **目的** | バウンディングスフィア内の Froxel にシャドウ済みローカルライトの散乱値を注入 |
| **レンダーターゲット** | `LightScattering`（Texture3D）の各 Z スライス |

---

## LightScatteringCS（:767）

### エントリポイント

```hlsl
[numthreads(THREADGROUP_SIZE_X, THREADGROUP_SIZE_Y, THREADGROUP_SIZE_Z)]
void LightScatteringCS(uint3 GridCoordinate : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Compute Shader |
| **スレッドグループ** | 8×8×1 |
| **目的** | ディレクショナルライト + スカイライト + Light Grid からの散乱を計算 + Temporal Reprojection |

#### 主要パーミュテーション

| マクロ | 説明 |
|-------|------|
| `LUMEN_GI` | Lumen GI の間接光を Froxel に注入 |
| `VIRTUAL_SHADOW_MAP` | VSM シャドウマスク使用 |
| `DISTANCE_FIELD_SKY_OCCLUSION` | DFAO によるスカイオクルージョン |
| `USE_RAYTRACED_SHADOWS` | ハードウェアレイトレーシングシャドウ |

#### CPU バインド

```cpp
// VolumetricFogRendering.cpp
FLightScatteringCS::FParameters* PassParameters = ...;
PassParameters->VolumetricFog = VolumetricFogUniformBuffer;
PassParameters->RWLightScattering = GraphBuilder.CreateUAV(LightScattering);
PassParameters->VBufferA = VBufferA;
PassParameters->VBufferB = VBufferB;
PassParameters->UnjitteredClipToTranslatedWorld = View.ClipToTranslatedWorld;
PassParameters->UnjitteredPrevTranslatedWorldToClip = View.PrevViewInfo.TranslatedWorldToClip;

FComputeShaderUtils::AddPass(
    GraphBuilder,
    RDG_EVENT_NAME("VolumetricFogLightScattering"),
    ComputeShader, PassParameters,
    FIntVector(GridSizeX, GridSizeY, GridSizeZ));
```

---

## FinalIntegrationCS（:1096）

### エントリポイント

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void FinalIntegrationCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Compute Shader |
| **スレッドグループ** | 8×8×1（XY のみ並列、Z は各スレッドがループ）|
| **目的** | Z スライスを積分して `IntegratedLightScattering` を生成 |

#### 出力バインド

| バインド名 | 型 | 説明 |
|-----------|------|------|
| `RWIntegratedLightScattering` | `RWTexture3D<float4>` | rgb=積分散乱色, a=積分透過率 |

---

## ComputeDepthFromZSlice（VolumetricFog.usf:58）

```hlsl
float ComputeDepthFromZSlice(float ZSlice)
{
    float SliceDepth = (exp2(ZSlice / VolumetricFog.GridZParams.z) - VolumetricFog.GridZParams.y)
                       / VolumetricFog.GridZParams.x;
    return SliceDepth;
}
// Z スライス → ワールド深度の変換（対数分布）
// CPU 側の ComputeZSliceFromDepth() の逆関数
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.VolumetricFog` | 1 | Volumetric Fog 有効 |
| `r.VolumetricFog.GridPixelSize` | 8 | Froxel XY ピクセルサイズ |
| `r.VolumetricFog.GridSizeZ` | 64 | Z スライス数 |
| `r.VolumetricFog.TemporalReprojection` | 1 | テンポラル再投影 |
| `r.VolumetricFog.HistoryMissSupersampleCount` | 4 | 履歴ミス時追加サンプル |
