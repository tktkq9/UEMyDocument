# d: MegaLights Resolve・デノイズ・TranslucencyVolume

- 対象ファイル: `MegaLightsResolve.cpp` / `MegaLightsDenoising.cpp`
- 概要: [[06_megalights_overview]]

---

## 概要

サンプリング→シャドウテスト後の LightSamples を BRDF 評価して SceneColor に合成するフェーズ。  
1. **Resolve**: サンプルの BRDF 評価 + 合算・正規化  
2. **Denoise**: テンポラル蓄積 + 空間フィルタでノイズ除去  
3. **TranslucencyVolume**: 半透明オブジェクト用の 3D ライティングボリューム生成

---

## Resolve フロー

```
FMegaLightsResolveCS（MegaLightsResolve.cpp）
  │
  ├─ LightSamples テクスチャを読み込み
  │   → 選択されたライストインデックス / 可視性 / 重みを取得
  │
  ├─ ForwardLightData からライスト情報（色・強度・位置）取得
  │
  ├─ GBuffer から BaseColor / Normal / Roughness を読み込み
  │   → BRDF（GGX + Lambertian）評価
  │
  ├─ Irradiance = Visibility × BRDF × LightIntensity / SamplePDF
  │   → PDF で除算して MIS（Multiple Importance Sampling）補正
  │
  └─ NumSamplesPerPixel 個の Irradiance を平均
      → ResolvedRadiance テクスチャに書き込み
```

---

## Denoising フロー

```
MegaLightsDenoising.cpp

[Temporal（テンポラル蓄積）]
  ├─ History テクスチャ（前フレームの Radiance）をリプロジェクション
  ├─ Neighborhood Clamp（近傍ピクセルの値でゴースト抑制）
  │   r.MegaLights.Temporal.NeighborhoodClampScale で強度調整
  ├─ HistoryLength（蓄積フレーム数）更新
  │   - History Miss → MinFramesAccumulated = 4（デフォルト）
  │   - 通常 → MaxFramesAccumulated = 12（デフォルト）
  └─ BlendFactor = 1 / HistoryLength でブレンド

[Spatial（空間フィルタ）]
  └─ r.MegaLights.Spatial=1 時に実行
      → 深度・法線ベースのバイラテラルフィルタ
      → r.MegaLights.Spatial.DepthWeightScale で調整
```

---

## TranslucencyVolume（半透明への照明）

```cpp
// MegaLightsVolume のデータ構造（MegaLights.h）
class FMegaLightsVolume
{
    FRDGTextureRef Texture;                          // 不透明ライティング
    FRDGTextureRef TranslucencyAmbient[TVC_MAX];     // 半透明アンビエント
    FRDGTextureRef TranslucencyDirectional[TVC_MAX]; // 半透明ディレクショナル
    // TVC_MAX = TranslucencyVolumeCascadeCount
};
```

```
[Volume注入フロー]
MegaLights::RayTraceLightSamples() のVolume版
  → VolumeLightSamples / VolumeLightSampleRays でシャドウテスト
  → FMegaLightsVolumeParameters でフレキセル（Froxel）単位処理
  → TranslucencyAmbient / TranslucencyDirectional テクスチャに書き込み
  → 半透明レンダリング時に TranslucentBasePassUniformParameters 経由で参照
```

---

## 主要 CVar（Denoise 関連）

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.MegaLights.Temporal` | true | テンポラル蓄積の有効/無効 |
| `r.MegaLights.Temporal.MaxFramesAccumulated` | 12 | 最大蓄積フレーム数（多いほどノイズ少・ゴースト多）|
| `r.MegaLights.Temporal.NeighborhoodClampScale` | 1.0 | 近傍クランプ強度（高いほどゴーストが出やすい）|
| `r.MegaLights.Spatial` | true | 空間フィルタの有効/無効 |
| `r.MegaLights.Volume` | 1 | TranslucencyVolume の有効/無効 |

---

## 関連リファレンス

- [[ref_megalights_denoising]] — `FMegaLightsDenoisingParameters` / ViewState 詳細
- [[a_megalights_pipeline]] — Resolve・Denoise の呼び出し位置（ステップ D・E・F）

---

## Resolve → Denoiser → Volume 注入 詳細フロー

```
【Resolve → SceneColor 加算までの詳細】

FMegaLightsResolveCS（MegaLightsResolve.cpp）
  per pixel（ダウンサンプル解像度）:
  │
  ├─ for s in [0, NumSamplesPerPixel):
  │   LightSamples[PixelX, PixelY, s]:
  │     LightIndex   = サンプルされたライスト番号
  │     Visibility   = 0.0（遮蔽）or 1.0（非遮蔽）
  │     SampleWeight = 1.0 / (PDF × NumSamples)
  │   │
  │   ├─ FLightData = ForwardLightData[LightIndex]
  │   ├─ GBuffer = GetGBufferData(ScreenUV)
  │   │
  │   ├─ DiffuseBRDF  = BaseColor/PI × max(NdotL, 0)
  │   ├─ SpecularBRDF = GGX_D × G × F / (4 × NdotV × NdotL)
  │   ├─   F0         = lerp(0.04, BaseColor, Metallic)
  │   │
  │   ├─ Irradiance[s] = Visibility × (DiffuseBRDF + SpecularBRDF)
  │   │                × LightData.Color × LightData.Attenuation
  │   │                × SampleWeight
  │   │   ※ firefly 抑制: Irradiance[s] = min(Irradiance[s], MaxShadingWeight)
  │   │
  │   └─ DiffuseAcc  += DiffuseBRDF  × Irradiance[s]
  │      SpecularAcc += SpecularBRDF × Irradiance[s]
  │
  ├─ ResolvedDiffuse  = DiffuseAcc  / NumSamplesPerPixel
  ├─ ResolvedSpecular = SpecularAcc / NumSamplesPerPixel
  └─ ResolvedRadiance テクスチャ（R11G11B10F × 2ch）に書き込み

【Temporal Denoiser 詳細】

FMegaLightsTemporalCS（MegaLightsDenoising.cpp）
  per pixel:
  │
  ├─ PrevScreenPos = ReprojectScreenPosition(ScreenPos, SceneDepth, PrevViewProj)
  ├─ History = DiffuseLightingHistory.SampleLevel(PrevScreenPos, 0)
  │             + SpecularLightingHistory.SampleLevel(PrevScreenPos, 0)
  │
  ├─ [Neighborhood Clamp]
  │   Moments1 = 3x3 近傍の ResolvedRadiance の平均（1次モーメント）
  │   Moments2 = 3x3 近傍の ResolvedRadiance² の平均（2次モーメント）
  │   Variance = Moments2 - Moments1²
  │   Sigma = sqrt(max(Variance, 0))
  │   ClampMin = Moments1 - NeighborhoodClampScale × Sigma
  │   ClampMax = Moments1 + NeighborhoodClampScale × Sigma
  │   History = clamp(History, ClampMin, ClampMax)
  │
  ├─ VisibleLightHash 比較:
  │   現フレーム Hash ≠ 前フレーム Hash → NumFrames = MinFramesAccumulated
  │   現フレーム Hash = 前フレーム Hash → NumFrames = min(NumFrames+1, MaxFrames)
  │
  ├─ BlendFactor = 1.0 / NumFrames
  ├─ Denoised = lerp(History, ResolvedRadiance, BlendFactor)
  └─ DiffuseLighting / SpecularLighting テクスチャに書き込み
     → QueueTextureExtraction() で ViewState.DiffuseLightingHistory に保存

【Spatial Filter 詳細（r.MegaLights.Spatial=1 時）】

FMegaLightsSpatialCS（MegaLightsDenoising.cpp）
  per pixel:
  │
  ├─ 5x5 または 7x7 のバイラテラルカーネル
  ├─ 重み = exp(-ΔDepth² × DepthWeightScale) × exp(-ΔNormal² × NormalWeightScale)
  └─ Filtered = sum(Denoised × Weight) / sum(Weight)
  → SceneColor RT（ELoad）に加算

【TranslucencyVolume 注入詳細（r.MegaLights.Volume=1 時）】

MegaLights::RayTraceLightSamples()（Volume 版）
  per voxel（FroxelGrid 3D）:
  │
  ├─ Froxel の中心ワールド座標を計算
  ├─ ClusteredLightGrid から Froxel 内のライスト一覧を取得
  ├─ PDF + BlueNoise でサンプリング
  ├─ シャドウテスト（EnabledRT: RayQuery / EnabledVSM: VSM 参照）
  ├─ Henyey-Greenstein 位相関数 (VolumePhaseG) でボリューム散乱を評価
  └─ TranslucencyAmbient[Cascade]    ← 等方性照明（SH0）
     TranslucencyDirectional[Cascade] ← 方向性照明（SH1）
     に書き込み（カスケードごとに2枚）

[半透明レンダリングでの参照]
TranslucentBasePassUniformParameters.MegaLightsVolume
  → TranslucencyVolumeShading.usf
  → Froxel インデックスから TranslucencyAmbient を SampleLevel()
  → 半透明マテリアルの Emissive / IndirectLighting に加算
```
