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

## FScreenProbeIntegrateParameters

統合パスに渡すパラメータ。ダウンサンプルした深度・法線情報と統合範囲を保持する。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FScreenProbeIntegrateParameters, )
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float>, DownsampledSceneDepth)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<UNORM float3>, DownsampledSceneWorldNormal)
    SHADER_PARAMETER(FIntPoint, IntegrateViewMin)
    SHADER_PARAMETER(FIntPoint, IntegrateViewSize)
    SHADER_PARAMETER(FVector2f, DownsampledBufferInvSize)
    SHADER_PARAMETER(uint32,   ScreenProbeGatherStateFrameIndex)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `DownsampledSceneDepth` | `Texture2D<float>` | ダウンサンプル済みシーン深度（統合パスが使う解像度）|
| `DownsampledSceneWorldNormal` | `Texture2D<UNORM float3>` | ダウンサンプル済みワールド法線 |
| `IntegrateViewMin` | `FIntPoint` | 統合対象ビュー領域の左上座標 |
| `IntegrateViewSize` | `FIntPoint` | 統合対象ビュー領域のサイズ |
| `DownsampledBufferInvSize` | `FVector2f` | ダウンサンプルバッファの逆解像度（UV 計算用）|
| `ScreenProbeGatherStateFrameIndex` | `uint32` | テンポラル用フレームインデックス（ジッター回転に使用）|

### 使用箇所

- [[ref_lumen_screen_probe_filtering]] — 統合（Integrate）CS に引数として渡される
- 統合パス（LumenScreenProbeIntegrate.cpp）— `IntegrateViewMin / Size` で処理範囲を決定

---

## FTileClassifyParameters

統合タイルの分類に使うパラメータ。  
タイル分類により VGPR が少ないタイルは軽量シェーダーで処理される。

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

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `DefaultDiffuseIntegrationMethod` | `uint32` | デフォルト統合方式（0=事前積分, 1=BRDF IS, 2=数値参照）|
| `MaxRoughnessToEvaluateRoughSpecular` | `float` | Rough Specular を評価する最大ラフネス値 |
| `MaxRoughnessToEvaluateRoughSpecularForFoliage` | `float` | フォリッジ向けの Rough Specular ラフネス上限 |
| `LumenHistoryDistanceThreshold` | `float` | テンポラル再投影の距離しきい値（超えたら履歴を破棄）|
| `LumenHistoryDistanceThresholdForFoliage` | `float` | フォリッジ向けの距離しきい値（緩め）|
| `LumenHistoryNormalCosThreshold` | `float` | 法線の cosθ しきい値（低すぎる場合は履歴を破棄）|

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — `LumenScreenProbeGather::SetupTileClassifyParameters()` で構築
- [[ref_lumen_screen_probe_filtering]] — タイル分類 CS に引数として渡され、`EScreenProbeIntegrateTileClassification` の決定に使用

---

## フィルタリングパイプライン

```
[トレース結果] FScreenProbeParameters.TraceRadiance / TraceHit
  │
  ├─ [空間フィルタ（SpatialFilter）]
  │   UseProbeSpatialFilter() = true の場合に実行
  │   └─ 近傍プローブの TraceRadiance を法線・深度で重み付けして平均化
  │       → ScreenProbeInterpolationDepthWeight で距離重みを制御
  │       → 法線の向きが大きく異なるプローブは除外（リーク防止）
  │       → ScreenProbeInterpolationDepthWeightForFoliage でフォリッジ調整
  │
  ├─ [テンポラルフィルタ（TemporalFilter）]
  │   UseProbeTemporalFilter() = true の場合に実行
  │   └─ 前フレームの ScreenProbeGatherParameters と現フレームを混合
  │       → ScreenProbeMoving テクスチャで動的オブジェクト領域を検出
  │       → 動いている領域はブレンド率を上げて追従性を向上
  │       → LumenHistoryDistanceThreshold / NormalCosThreshold で履歴検証
  │
  └─ [SH 射影（Project to SH）]
        照度を球面調和関数（L1 SH）に射影
        → ScreenProbeRadianceSHAmbient（環境成分）
        → ScreenProbeRadianceSHDirectional（指向成分）
        ─ IrradianceFormat = Octahedral の場合は SH をスキップ
        → FScreenProbeGatherParameters に結果を書き込み
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

## ScreenProbeMoving テクスチャの役割

```
ScreenProbeMoving (Texture2D<float>):
  各プローブが「動いているか」を 0.0〜1.0 で示すテクスチャ

  → ScreenProbeWorldSpeed（Motion Vector）から動的判定
  → 速度差が RelativeSpeedDifferenceToConsiderLightingMoving を超えると 1.0 に近づく
  → 1.0 に近いほど照明が変化しやすいとみなして
    テンポラルブレンド率を上げる（テンポラルラグを抑制）

  → 静的なシーンでは 0.0 → 多くのフレームを蓄積してノイズを削減
  → 動的オブジェクト周囲では高い値 → 素早く現フレームに追従
```

### 使用箇所

- [[ref_lumen_screen_probe_filtering]] — テンポラルフィルタのブレンド率決定に使用
- [[ref_lumen_screen_probe_gather]] — `FScreenProbeGatherParameters::ScreenProbeMoving` として格納

---

## EScreenProbeIntegrateTileClassification との連携

```
タイル分類 → 最適なシェーダーの選択:

  SimpleDiffuse:
    → BRDF IS 不要（完全拡散マテリアルのみ）
    → 最も VGPR が少ない軽量シェーダー

  SupportImportanceSampleBRDF:
    → 高ラフネス鏡面を含むタイル
    → StructuredImportanceSampledRayInfosForTracing を参照する IS シェーダー

  SupportAll:
    → 低ラフネス鏡面も含むタイル（最大 VGPR）
    → 鏡面 BRDF 評価 + IS の完全シェーダー

分類方法:
  各タイルの GBuffer.Roughness の最小値 → タイル分類を決定
  → SimpleDiffuse タイルはコストを大幅削減できる
```
