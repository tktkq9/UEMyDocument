# GPU b: Denoising — BRDF シェーディング + テンポラル/空間デノイズ

- シェーダー: `MegaLights/MegaLightsShading.usf`, `MegaLightsDenoiserTemporal.usf`, `MegaLightsDenoiserSpatial.usf`
- CPU 対応: [[d_megalights_resolve]]
- 上位: [[01_megalights_gpu_overview]]

---

## 概要

サンプリング・可視性テスト後の LightSamples を 3 段階で処理する:

1. **ShadeLightSamplesCS** — BRDF（GGX + Lambertian）評価 → Resolved 輝度を計算
2. **DenoiserTemporalCS** — テンポラル蓄積・Neighborhood Clamp でゴースト抑制
3. **DenoiserSpatialCS** — 空間バイラテラルフィルタ → 最終 SceneColor に加算

---

## ShadeLightSamplesCS（MegaLightsShading.usf: line 166）

ダウンサンプル解像度で動作し、各ピクセルのサンプルを BRDF 評価して合算する。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ShadeLightSamplesCS(
    uint3 GroupId : SV_GroupID,
    uint3 GroupThreadId : SV_GroupThreadID,
    uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // 1. TileData から処理タイル座標を取得
    // 2. LoadMaterial() で GBuffer を読み込み
    // 3. INPUT_TYPE == INPUT_TYPE_GBUFFER の場合 LightingChannelMask を取得
    // 4. DOWNSAMPLE_FACTOR に応じてダウンサンプルサンプル重みを計算
    // 5. for each Sample in [0, NUM_SAMPLES_PER_PIXEL_1D):
    //    LightSamples から LightIndex / Visibility / SampleWeight を取得
    //    ForwardLightData[LightIndex] から色・強度を取得
    //    DiffuseBRDF = BaseColor/PI × NdotL
    //    SpecularBRDF = GGX_D × G × F / (4 × NdotV × NdotL)
    //    Irradiance = Visibility × BRDF × LightColor × Attenuation × SampleWeight
    //    Irradiance = min(Irradiance, MaxShadingWeight)  // firefly 抑制
    // 6. 全サンプルの平均 → ResolvedDiffuse / ResolvedSpecular に書き込み
}
```

**パーミュテーション:**

| マクロ | 値 | 説明 |
|--------|-----|------|
| `TILE_TYPE` | 0–12 | `TILE_MODE_*` |
| `INPUT_TYPE` | `INPUT_TYPE_GBUFFER` / `INPUT_TYPE_HAIRSTRANDS` | 入力ソース |
| `DOWNSAMPLE_FACTOR_X` / `Y` | 1, 2 | ダウンサンプル率 |
| `NUM_SAMPLES_PER_PIXEL_1D` | 1/2/4/16 | サンプル数 |

---

## ClearResolvedLightingCS（MegaLightsShading.usf: line 454）

フレーム開始時に Resolved テクスチャをクリアする補助 CS。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ClearResolvedLightingCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

---

## DenoiserTemporalCS（MegaLightsDenoiserTemporal.usf: line 183）

テンポラル蓄積と Neighborhood Clamp によるゴースト抑制を行う。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void DenoiserTemporalCS(
    uint3 GroupId : SV_GroupID,
    uint3 GroupThreadId : SV_GroupThreadID,
    uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // 1. ReprojectScreenPosition() で前フレーム UV を計算
    // 2. DiffuseLightingHistory / SpecularLightingHistory をサンプリング
    // 3. Neighborhood Clamp:
    //    3x3 近傍の Moments1（平均）と Moments2（二乗平均）を計算
    //    Sigma = sqrt(Moments2 - Moments1^2)
    //    ClampMin = Moments1 - NeighborhoodClampScale × Sigma
    //    ClampMax = Moments1 + NeighborhoodClampScale × Sigma
    //    History = clamp(History, ClampMin, ClampMax)
    // 4. VisibleLightHash 比較:
    //    Hash 変化 → NumFrames = MinFramesAccumulated（デフォルト 4）
    //    Hash 一致 → NumFrames = min(NumFrames + 1, MaxFramesAccumulated)
    // 5. BlendFactor = 1.0 / NumFrames
    //    Denoised = lerp(History, ResolvedRadiance, BlendFactor)
    // 6. RWDiffuseLighting / RWSpecularLighting に書き込み
    //    ViewState.DiffuseLightingHistory に保存（QueueTextureExtraction）
}
```

**入力テクスチャ:**

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `ResolvedDiffuseLighting` | `Texture2D<half3>` | ShadeLightSamplesCS の出力（拡散）|
| `ResolvedSpecularLighting` | `Texture2D<half3>` | ShadeLightSamplesCS の出力（鏡面）|
| `DiffuseLightingHistoryTexture` | `Texture2D<half3>` | 前フレーム拡散 History |
| `SpecularLightingHistoryTexture` | `Texture2D<half3>` | 前フレーム鏡面 History |
| `LightingMomentsHistoryTexture` | `Texture2D<half4>` | 前フレームの分散情報（Moments）|
| `NumFramesAccumulatedHistoryTexture` | `Texture2D<half>` | 蓄積フレーム数 |
| `MegaLightsDepthHistory` | `Texture2D` | 前フレーム深度（リプロジェクション判定）|
| `MegaLightsNormalAndShading` | `Texture2D<float4>` | 法線 + シェーディング信頼度 |
| `EncodedHistoryScreenCoordTexture` | `Texture2D<uint>` | エンコード済み前フレーム座標 |
| `PackedPixelDataTexture` | `Texture2D<uint>` | パックされたピクセルデータ |

**スカラーパラメータ:**

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `TemporalMaxFramesAccumulated` | `float` | 最大蓄積フレーム数（デフォルト 12）|
| `TemporalNeighborhoodClampScale` | `float` | Neighborhood Clamp 強度（デフォルト 1.0）|
| `MinFramesAccumulatedForHistoryMiss` | `float` | History ミス時の最小蓄積数（デフォルト 4）|
| `PrevSceneColorPreExposureCorrection` | `float` | 露出補正係数 |

---

## DenoiserSpatialCS（MegaLightsDenoiserSpatial.usf: line 47）

分散ベースの適応空間フィルタで残留ノイズを除去し、最終的に SceneColor に加算する。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void DenoiserSpatialCS(
    uint3 GroupId : SV_GroupID,
    uint3 GroupThreadId : SV_GroupThreadID,
    uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // 1. LightingMoments から CenterDiffuseStdDev を計算
    // 2. ShadingConfidence <= 0.25 → DisocclusionFactor を計算
    // 3. [SPATIAL_FILTER]
    //    Hammersley16() + UniformSampleDiskConcentric() でランダム近傍を生成
    //    深度平面距離差 → DepthWeight = exp2(-DepthWeightScale × RelDepthDiff^2)
    //    法線差 → NormalWeight = 1 - acos(dot(N1, N2))
    //    輝度差 → LuminanceWeight
    //    Weight = DepthWeight × NormalWeight × LuminanceWeight
    //    フィルタ適用: DiffuseLighting = Σ(Lighting × Weight) / Σ(Weight)
    // 4. GetDenoisingModulateFactors() で Diffuse/Specular を変調（Substrate 対応）
    // 5. LightAccumulator_AddSplit() で SceneColor に加算
    //    RWSceneColor[ScreenCoord] += AccumulatedSceneColor
}
```

**入力テクスチャ:**

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `DiffuseLightingTexture` | `Texture2D<float3>` | テンポラル後の拡散光 |
| `SpecularLightingTexture` | `Texture2D<float3>` | テンポラル後の鏡面光 |
| `LightingMomentsTexture` | `Texture2D<float4>` | 分散情報（Diffuse + Specular Moments）|
| `NumFramesAccumulatedTexture` | `Texture2D<UNORM float>` | 蓄積フレーム数 |
| `ShadingConfidenceTexture` | `Texture2D<UNORM float>` | シェーディング信頼度 |

**出力:**

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `RWSceneColor` | `RWTexture2D<float4>` | 最終合成先（加算ブレンド）|

**スカラーパラメータ:**

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `SpatialFilterDepthWeightScale` | `float` | 深度重みのスケール |
| `SpatialFilterKernelRadius` | `float` | カーネル半径（ピクセル単位）|
| `SpatialFilterNumSamples` | `uint` | サンプル数 |
| `SpatialFilterMaxDisocclusionFrames` | `float` | Disocclusion 時の最大フレーム数 |

**パーミュテーション:**

| マクロ | 値 | 説明 |
|--------|-----|------|
| `INPUT_TYPE` | `INPUT_TYPE_GBUFFER` / `INPUT_TYPE_HAIRSTRANDS` | 入力ソース |
| `SPATIAL_FILTER` | 0/1 | 空間フィルタの有効/無効 |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.MegaLights.Temporal` | true | テンポラル蓄積の有効/無効 |
| `r.MegaLights.Temporal.MaxFramesAccumulated` | 12 | 最大蓄積フレーム数 |
| `r.MegaLights.Temporal.NeighborhoodClampScale` | 1.0 | Neighborhood Clamp 強度 |
| `r.MegaLights.Spatial` | true | 空間フィルタの有効/無効 |
| `r.MegaLights.Spatial.DepthWeightScale` | （内部）| 深度重みスケール |
| `r.MegaLights.MaxShadingWeight` | 20.0 | ファイアフライ抑制クランプ |
