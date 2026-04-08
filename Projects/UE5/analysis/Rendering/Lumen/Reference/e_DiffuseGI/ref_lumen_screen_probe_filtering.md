# リファレンス：LumenScreenProbeFiltering.cpp

- グループ: e - Diffuse GI
- 上位: [[e_lumen_diffuse_gi]]
- 関連: [[ref_lumen_screen_probe_gather]] | [[ref_lumen_screen_probe_tracing]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenScreenProbeFiltering.cpp`

---

## 概要

Screen Probe のトレース結果を**空間フィルタリング**および**テンポラル蓄積**して  
ノイズを除去し、滑らかな照度テクスチャを生成するファイル。  
`FScreenProbeGatherParameters` に書き込んだ結果が後続の統合パスで使われる。

---

## フィルタリングパイプライン

```
[トレース結果] FScreenProbeParameters.TraceRadiance / TraceHit
  │
  ├─ [空間フィルタ（SpatialFilter）]
  │   UseProbeSpatialFilter() = true の場合に実行
  │   └─ 近傍プローブの TraceRadiance を法線・深度で重み付けして平均化
  │       → 法線の向きが大きく異なるプローブは除外（リーク防止）
  │       → ScreenProbeInterpolationDepthWeight で距離重みを制御
  │
  ├─ [テンポラルフィルタ（TemporalFilter）]
  │   UseProbeTemporalFilter() = true の場合に実行
  │   └─ 前フレームの ScreenProbeGatherParameters と現フレームを混合
  │       → ScreenProbeMoving テクスチャで動的オブジェクト領域を検出
  │       → 動いている領域はブレンド率を上げて追従性を向上
  │
  └─ [SH 射影（Project to SH）]
        照度を球面調和関数（L1 SH）に射影
        → ScreenProbeRadianceSHAmbient（環境成分）
        → ScreenProbeRadianceSHDirectional（指向成分）
        ─ IrradianceFormat = Octahedral の場合は SH をスキップ
```

---

## FScreenProbeIntegrateParameters

統合パスに渡すパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FScreenProbeIntegrateParameters, )
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float>, DownsampledSceneDepth)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<UNORM float3>, DownsampledSceneWorldNormal)
    SHADER_PARAMETER(FIntPoint, IntegrateViewMin)
    SHADER_PARAMETER(FIntPoint, IntegrateViewSize)
    SHADER_PARAMETER(FVector2f, DownsampledBufferInvSize)
    SHADER_PARAMETER(uint32,   ScreenProbeGatherStateFrameIndex) // テンポラル用フレームインデックス
END_SHADER_PARAMETER_STRUCT()
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.ScreenProbeGather.SpatialFilterProbes` | 1 | 空間フィルタの有効/無効 |
| `r.Lumen.ScreenProbeGather.SpatialFilterMaxRadianceHitAngle` | — | 空間フィルタ最大ヒット角度 |
| `r.Lumen.ScreenProbeGather.Temporal` | 1 | テンポラルフィルタの有効/無効 |
| `r.Lumen.ScreenProbeGather.Temporal.MaxFramesAccumulated` | — | テンポラル最大蓄積フレーム数 |
| `r.Lumen.ScreenProbeGather.InterpolationDepthWeight` | 1.0 | 深度重み（フィルタのシャープネス）|
| `r.Lumen.ScreenProbeGather.InterpolationDepthWeightForFoliage` | 0.25 | フォリッジ用の深度重み |
| `r.Lumen.ScreenProbeGather.FullResolutionJitterWidth` | 1.0 | フルレゾリューション ジッター幅 |

---

## FTileClassifyParameters

統合タイルの分類に使うパラメータ。タイル分類により VGPR が少ないタイルは軽量シェーダーで処理される。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FTileClassifyParameters, )
    SHADER_PARAMETER(uint32, DefaultDiffuseIntegrationMethod)
    SHADER_PARAMETER(float,  MaxRoughnessToEvaluateRoughSpecular)
    SHADER_PARAMETER(float,  MaxRoughnessToEvaluateRoughSpecularForFoliage)
    SHADER_PARAMETER(float,  LumenHistoryDistanceThreshold)
    SHADER_PARAMETER(float,  LumenHistoryDistanceThresholdForFoliage)
    SHADER_PARAMETER(float,  LumenHistoryNormalCosThreshold)
END_SHADER_PARAMETER_STRUCT()
```

---

## ScreenProbeMoving テクスチャの役割

```
ScreenProbeMoving (Texture2D<float>):
  各プローブが「動いているか」を 0.0〜1.0 で示すテクスチャ
  
  → 速度ベクトル（ScreenProbeWorldSpeed）から動的判定
  → 1.0 に近いほど照明が変化しやすいとみなして
    テンポラルブレンド率を上げる（テンポラルラグを抑制）
  
  → 静的なシーンでは 0.0 → 多くのフレームを蓄積してノイズを削減
```
