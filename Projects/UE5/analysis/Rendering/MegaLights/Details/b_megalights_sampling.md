# b: 確率的ライトサンプリング

- 対象ファイル: `MegaLightsSampling.cpp`
- 概要: [[06_megalights_overview]]

---

## 概要

**確率的（Stochastic）サンプリング**により、多数のライトから少数のサンプルを  
選択してシェーディングコストを O(N) → O(1) に近づける。  
各ピクセルに対して `r.MegaLights.NumSamplesPerPixel`（デフォルト4）個のライトをランダムに選択し  
その重要度に比例した確率でサンプリングする。

---

## FGenerateLightSamplesCS

```cpp
// MegaLightsSampling.cpp:59
class FGenerateLightSamplesCS : public FGlobalShader
{
    DECLARE_GLOBAL_SHADER(FGenerateLightSamplesCS);
    SHADER_USE_PARAMETER_STRUCT(FGenerateLightSamplesCS, FGlobalShader);

    // 出力: LightSamples テクスチャ（各ピクセルのサンプルされたライスト情報）
    // 出力: LightSampleRays テクスチャ（シャドウレイの方向・距離）
};
```

---

## サンプリング戦略

```
各ピクセルでの処理（FGenerateLightSamplesCS）:
  │
  ├─ [A] ライスト構築
  │   → ForwardLightData バッファから可視ライスト一覧を取得
  │   → ClusteredLightGrid[ClusterKey] → このピクセルに影響するライスト
  │
  ├─ [B] 重要度サンプリング PDF 計算
  │   → 各ライストの Irradiance（強度 × 立体角）を推定
  │   → [r.MegaLights.GuideByHistory > 0]
  │       前フレームの Shadow 結果でPDFを補正
  │       → 見えていたライトは高PDF / 遮蔽ライストは低PDF
  │
  ├─ [C] サンプル選択（r.MegaLights.NumSamplesPerPixel 個）
  │   → Blue Noise を利用した低ディスコーレパンシーサンプリング
  │   → 選択ライストの Index, Weight, Direction を LightSamples に書き込み
  │
  └─ [D] Ray 生成
      → 各サンプルに対してシャドウレイ（Origin, Direction, TMax）を生成
      → LightSampleRays テクスチャに書き込み
```

---

## ダイレクショナルライトの特別扱い

```cpp
// MegaLightsSampling.cpp
// DirectionalLight は通常のライストに混在させず、専用の比率でサンプリング
float GetDirectionalLightSampleRatio()
{
    float Fraction = CVarMegaLightsDirectionalLightSampleFraction.GetValueOnRenderThread();
    // Fraction = 0.5 → DirectionalLight : OtherLights = 1:1
    if (Fraction < 1.0f)
        return Fraction / (1.0f - Fraction);
    else
        return 0.0f;
}
```

---

## FMegaLightsParameters（シェーダーへの入力）

```cpp
// MegaLightsInternal.h:11
BEGIN_SHADER_PARAMETER_STRUCT(FMegaLightsParameters, )
    SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, ViewUniformBuffer)
    SHADER_PARAMETER_STRUCT_INCLUDE(FSceneTextureParameters, SceneTextures)
    SHADER_PARAMETER_STRUCT_REF(FBlueNoise, BlueNoise)         // ジッターサンプリング用
    SHADER_PARAMETER(FIntPoint, NumSamplesPerPixel)
    SHADER_PARAMETER(FIntPoint, DownsampleFactor)              // (2,2) for half-res
    SHADER_PARAMETER(float, MinSampleWeight)                   // r.MegaLights.MinSampleWeight
    SHADER_PARAMETER(float, MaxShadingWeight)                  // firefly clamp
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float>, DownsampledSceneDepth)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<UNORM float3>, DownsampledSceneWorldNormal)
    // ...
END_SHADER_PARAMETER_STRUCT()
```

---

## 主要 CVar（サンプリング関連）

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.MegaLights.NumSamplesPerPixel` | 4 | ピクセルあたりのサンプル数（2/4/16）|
| `r.MegaLights.DownsampleMode` | 2 | ダウンサンプル方式（0=なし/1=チェッカー/2=半解像度）|
| `r.MegaLights.MinSampleWeight` | 0.001 | 最小サンプル重み（小さいと影響なしでスキップ）|
| `r.MegaLights.MaxShadingWeight` | 20.0 | ファイアフライ抑制クランプ値 |
| `r.MegaLights.GuideByHistory` | 2 | History ガイド（0=無効/1=ライスト/2=ライスト部位）|

---

## 関連リファレンス

- [[ref_megalights_sampling]] — `FGenerateLightSamplesCS` / `FMegaLightsParameters` 詳細
- [[a_megalights_pipeline]] — パイプライン全体でのサンプリング位置

---

## GetMegaLightsMode() 判定 + ライスト構築 詳細フロー

```
【ライスト対象判定フロー】
  FSortedLightSceneInfo 構築時（GatherLightsAndComputeLightGrid）:
    │
    └─ for each FLightSceneInfo:
        SortKey.Fields.LightType      = PointLight / SpotLight / RectLight
        GetLightOcclusionType():
          └─ GetMegaLightsMode(ViewFamily, LightType, bAllowMegaLights, ShadowMethod)
              │
              ├─ !IsEnabled(ViewFamily) || !bAllowMegaLights
              │   → EMegaLightsMode::Disabled （SortedLights の通常スロットへ）
              │
              ├─ UseHardwareRayTracing(ViewFamily) && IsHardwareRayTracingSupported()
              │   → EMegaLightsMode::EnabledRT
              │
              └─ その他
                  → EMegaLightsMode::EnabledVSM
        │
        ├─ Disabled  → SortedLights[UnbatchedLightStart..] の通常スロットへ分類
        ├─ EnabledRT → SortedLights[MegaLightsLightStart..] へ分類
        └─ EnabledVSM→ SortedLights[MegaLightsLightStart..] へ分類 + VSM ページマーキング対象

【ライスト候補リスト構築フロー（FGenerateLightSamplesCS）】
  │
  ├─ ForwardLightUniformBuffer から NumLocalLights を取得
  ├─ ClusteredLightGrid[TileX, TileY, ClusterZ] → このピクセルのライスト候補インデックス
  │   ※ ClusteredLightGrid は GatherLightsAndComputeLightGrid() 時に構築済み
  │
  ├─ for each LightIndex in 候補リスト:
  │   [フィルタリング]
  │   ├─ FLightSceneInfo.MegaLightsLightStart 以降か判定
  │   ├─ LightingChannelMask & Object.LightingChannelMask != 0 チェック
  │   └─ FarField GDF 対象（r.MegaLights.FarField=1）かどうかチェック
  │
  ├─ 重要度（PDF）計算:
  │   EstimatedIrradiance = LightIntensity
  │                       × saturate(dot(Normal, LightDir))
  │                       × DistanceAttenuation(Distance, InvRadius)
  │   PDF[LightIndex] = EstimatedIrradiance / sum(EstimatedIrradiance for all candidates)
  │
  └─ [r.MegaLights.GuideByHistory > 0]
      VisibleLightHashHistory から前フレームの可視ライスト情報を取得
      → 前フレームに Visibility=1 だったライスト: PDF を引き上げ
      → 前フレームに Visibility=0（遮蔽）だったライスト: PDF を引き下げ
      → 再正規化して最終 PDF を決定

【サンプリング実行】
  ※ MIS (Multiple Importance Sampling) + RIS (Resampled Importance Sampling)
  BlueNoise テクスチャ[PixelX, PixelY, FrameIndex % BlueNoiseSliceCount]
    → u = BlueNoise.SampleLevel(UV, 0)
    → CDF を u でバイナリサーチ
    → LightIndex[s] = 対応するライスト
    → SampleWeight[s] = 1.0 / (PDF[LightIndex[s]] × NumSamplesPerPixel)
```
