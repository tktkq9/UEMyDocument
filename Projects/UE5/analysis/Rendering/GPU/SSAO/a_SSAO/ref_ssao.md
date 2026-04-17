# REF: SSAO / GTAO シェーダー

- グループ: a - SSAO
- 詳細: [[detail_ssao]]
- CPU リファレンス: [[ref_ssao]]
- ソース: `Engine/Shaders/Private/PostProcessAmbientOcclusion.usf`

---

## MainSetupPS（:190）

```hlsl
void MainSetupPS(
    in noperspective float4 UVAndScreenPos : TEXCOORD0,
    in FStereoPSInput StereoInput,
    float4 SvPosition : SV_POSITION,
    out float4 OutColor0 : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **目的** | SceneDepth と GBuffer Normal を半解像度に縮小 |
| **出力** | `OutColor0 = float4(Depth, Normal.xy, 0)` |

---

## MainPS（:489）/ MainCS（:498）

```hlsl
void MainPS(in noperspective float4 UVAndScreenPos : TEXCOORD0, ..., out float4 OutColor : SV_Target0)
void MainCS(uint2 PixelPos : SV_DispatchThreadID, ...)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 従来 SSAO（AO_SAMPLE_QUALITY = 1〜3）の半球サンプリング |
| **パーミュテーション** | `AO_SAMPLE_QUALITY`, `USE_AO_SETUP_AS_INPUT`, `USE_UPSAMPLE`, `SHADER_QUALITY` |

---

## HorizonSearchCS（:1165）/ HorizonSearchPS（:1156）

```hlsl
[numthreads(THREADGROUP_SIZEX, THREADGROUP_SIZEY, 1)]
void HorizonSearchCS(uint2 PixelPos : SV_DispatchThreadID)

void HorizonSearchPS(in noperspective float4 UVAndScreenPos : TEXCOORD0, ..., out float4 OutColor : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **目的** | GTAO の Horizon Angle 検索（複数方向 × SAMPLE_STEPS）|
| **出力** | `float4(HorizonAngle1, HorizonAngle2, BentNormal.x, BentNormal.y)` |

---

## GTAOInnerIntegrateCS（:1058）/ GTAOInnerIntegratePS（:1048）

```hlsl
[numthreads(THREADGROUP_SIZEX, THREADGROUP_SIZEY, 1)]
void GTAOInnerIntegrateCS(uint2 PixelPos : SV_DispatchThreadID)

void GTAOInnerIntegratePS(in noperspective float4 UVAndScreenPos : TEXCOORD0, out float4 OutColor : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Horizon Angle から AO 値を積分。Bent Normal を計算 |
| **出力** | `OutColor.r = AO`（R8）または `OutColor.rg = float2(AO, BentNormal)`（R8G8）|

---

## GTAOCombinedCS（:966）/ GTAOCombinedPS（:957）

```hlsl
[numthreads(THREADGROUP_SIZEX, THREADGROUP_SIZEY, 1)]
void GTAOCombinedCS(uint2 PixelPos : SV_DispatchThreadID)

void GTAOCombinedPS(in float4 UVAndScreenPos : TEXCOORD0, out float OutColor : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **目的** | HorizonSearch + Integrate を 1 パスに統合（`EGTAOType::EAsyncCombinedSpatial`）|
| **呼び出し条件** | `EGTAOType = EAsyncCombinedSpatial` または `EAsyncHorizonSearch`（Spatial filter 込み）|

---

## GTAOSpatialFilterCS（:1424）/ GTAOSpatialFilterPS（:1590）

```hlsl
[numthreads(THREADGROUP_SIZEX, THREADGROUP_SIZEY, 1)]
void GTAOSpatialFilterCS(uint2 PixelPos : SV_DispatchThreadID)

void GTAOSpatialFilterPS(float4 UVAndScreenPos : TEXCOORD0, ...)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 法線・深度 aware バイラテラルフィルタ（ノイズ低減）|
| **ブラーモード** | `QUAD_MESSAGE_PASSING_BLUR = 0/2/3` |

---

## GTAOTemporalFilterCS（:1353）/ GTAOTemporalFilterPS（:1342）

```hlsl
[numthreads(THREADGROUP_SIZEX, THREADGROUP_SIZEY, 1)]
void GTAOTemporalFilterCS(uint2 PixelPos : SV_DispatchThreadID)

void GTAOTemporalFilterPS(in noperspective float4 UVAndScreenPos : TEXCOORD0, ...)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 前フレーム AO を Velocity で再投影してブレンド（テンポラル蓄積）|
| **有効条件** | `EGTAOPass::TemporalFilter` フラグが立っている時 |

---

## GTAOUpsamplePS（:182）

```hlsl
void GTAOUpsamplePS(
    in noperspective float4 UVAndScreenPos : TEXCOORD0,
    out float4 OutColor : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 半解像度 AO テクスチャを全解像度にバイラテラルアップサンプル |
| **品質** | `AO_UPSAMPLE_QUALITY=0`（4 サンプル）/ `=1`（9 サンプル）|

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.AmbientOcclusion.Method` | 1 | 0=SSAO / 1=GTAO |
| `r.GTAO.UseNormals` | 1 | GBuffer 法線を使う |
| `r.GTAO.ThicknessBlend` | 0.5 | 厚さヒューリスティック |
| `r.GTAO.TemporalFilter` | 1 | テンポラルフィルタ有効 |
| `r.AmbientOcclusion.AsyncCompute` | 1 | AsyncCompute 有効 |
