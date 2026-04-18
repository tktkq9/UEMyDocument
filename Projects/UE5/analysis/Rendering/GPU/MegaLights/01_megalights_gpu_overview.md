# GPU MegaLights シェーダー全体概要

- CPU 対応: [[06_megalights_overview]]
- 上位: [[01_gpu_overview]]

---

## MegaLights GPU パイプライン

多数のダイナミックライトを **確率的サンプリング → 可視性テスト → シェーディング → デノイズ → 合成** の順に処理する。

```
【A】 タイル分類
  MegaLights.usf: MegaLightsTileClassificationBuildListsCS
  → スクリーン空間タイルを TILE_MODE 別に分類
  → DownsampledTileAllocator / TileAllocator に書き込み

【B】 ライトサンプリング
  MegaLightsSampling.usf: GenerateLightSamplesCS
  → BlueNoise + 重要度サンプリングで各ピクセルにライトを割り当て
  → LightSamples / LightSampleRays テクスチャを出力

【C】 可視性テスト（シャドウ）
  MegaLightsRayTracing.usf: ScreenSpaceRayTraceLightSamplesCS / SoftwareRayTraceLightSamplesCS
  MegaLightsHardwareRayTracing.usf: HW Ray Tracing (RayGen shader)
  → LightSamples.Visibility を更新

【D】 シェーディング（Resolve）
  MegaLightsShading.usf: ShadeLightSamplesCS
  → GBuffer から BRDF 評価 → ResolvedDiffuse / ResolvedSpecular に出力

【E】 デノイズ
  MegaLightsDenoiserTemporal.usf: DenoiserTemporalCS
  MegaLightsDenoiserSpatial.usf: DenoiserSpatialCS
  → Temporal 蓄積 + 空間フィルタ → SceneColor に加算

【F】 ボリューム（半透明用）
  MegaLightsVolumeSampling.usf: VolumeGenerateLightSamplesCS
  MegaLightsVolumeRayTracing.usf: VolumeCompactLightSampleTracesCS / VolumeSoftwareRayTraceLightSamplesCS
  MegaLightsVolumeShading.usf: VolumeShadeLightSamplesCS
  → TranslucencyAmbient / TranslucencyDirectional 3D テクスチャに照明を書き込み
```

---

## シェーダーグループ一覧

| グループ | ファイル | 主な CS | 役割 |
|--------|--------|--------|------|
| **a: LightSampling** | `MegaLightsSampling.usf`, `MegaLightsRayTracing.usf`, `MegaLightsHardwareRayTracing.usf` | `GenerateLightSamplesCS`, `ScreenSpaceRayTraceLightSamplesCS` | サンプリング + 可視性テスト |
| **b: Denoising** | `MegaLightsShading.usf`, `MegaLightsDenoiserTemporal.usf`, `MegaLightsDenoiserSpatial.usf` | `ShadeLightSamplesCS`, `DenoiserTemporalCS`, `DenoiserSpatialCS` | BRDF 評価 + デノイズ → SceneColor 合成 |
| **c: Composite** | `MegaLights.usf`, `MegaLightsVolumeSampling.usf`, `MegaLightsVolumeShading.usf`, `MegaLightsVolumeRayTracing.usf` | `MegaLightsTileClassificationBuildListsCS`, `VolumeGenerateLightSamplesCS`, `VolumeShadeLightSamplesCS` | タイル分類 + ボリューム照明 |

---

## 主要バッファ・テクスチャ

| リソース | 型 | 内容 |
|--------|-----|------|
| `LightSamples` | Texture2DArray | 各ピクセルのサンプルされたライト ID・重み・可視性 |
| `LightSampleRays` | Texture2DArray | シャドウレイの方向・距離 |
| `ResolvedDiffuseLighting` | Texture2D `half3` | BRDF 評価済み拡散光 |
| `ResolvedSpecularLighting` | Texture2D `half3` | BRDF 評価済み鏡面光 |
| `DiffuseLightingHistory` | Texture2D `half3` | テンポラル蓄積済み拡散光 |
| `TranslucencyAmbient[TVC_MAX]` | Texture3D | 半透明用アンビエント SH ボリューム |
| `TranslucencyDirectional[TVC_MAX]` | Texture3D | 半透明用ディレクショナル SH ボリューム |

---

## 動作モード

| モード | 可視性テスト | 説明 |
|-------|-----------|------|
| `Disabled` | — | MegaLights 無効（通常 Deferred）|
| `EnabledVSM` | VSM 参照 | Virtual Shadow Maps 経由 |
| `EnabledRT` | HW Ray Query | ハードウェアレイトレーシング |

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `MegaLights.ush` | 共通型・定数（FMegaLightsData 等）|
| `MegaLightsMaterial.ush` | マテリアル読み込み（LoadMaterial, FMegaLightsMaterial）|
| `MegaLightsSampling.ush` | サンプリング関数（FLightSampler 等）|
| `MegaLightsRayTracing.ush` | レイトレーシング共通関数 |
| `MegaLightsTileClassification.ush` | タイル分類定数・関数 |
| `MegaLightsVolume.ush` | ボリューム座標変換 |
| `/Engine/Shared/MegaLightsDefinitions.h` | `TILE_MODE_*` 定数定義 |
