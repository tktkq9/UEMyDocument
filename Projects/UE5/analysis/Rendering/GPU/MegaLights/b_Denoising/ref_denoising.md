# GPU b Ref: Denoising シェーダーリファレンス

- シェーダー: `MegaLights/MegaLightsShading.usf`, `MegaLightsDenoiserTemporal.usf`, `MegaLightsDenoiserSpatial.usf`
- CPU 対応: [[d_megalights_resolve]]
- 上位: [[01_megalights_gpu_overview]]

---

## エントリポイント一覧

### MegaLightsShading.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `ShadeLightSamplesCS` | 166 | CS | BRDF 評価・サンプル合算（Resolve）|
| `ClearResolvedLightingCS` | 454 | CS | Resolved テクスチャのクリア |

### MegaLightsDenoiserTemporal.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `GetNeighborhood` | 118 | helper | 3×3 近傍 Moments 計算 |
| `DenoiserTemporalCS` | 183 | CS | テンポラル蓄積 + Neighborhood Clamp |

### MegaLightsDenoiserSpatial.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `DenoiserSpatialCS` | 47 | CS | 空間バイラテラルフィルタ + SceneColor 合成 |

---

## ShadeLightSamplesCS パーミュテーション

| マクロ | 値 | 説明 |
|--------|-----|------|
| `TILE_TYPE` | 0–12 | `TILE_MODE_SIMPLE_SHADING` 〜 `TILE_MODE_COMPLEX_SPECIAL_SHADING_RECT_TEXTURED` |
| `INPUT_TYPE` | `INPUT_TYPE_GBUFFER` / `INPUT_TYPE_HAIRSTRANDS` | 入力ソース |
| `DOWNSAMPLE_FACTOR_X` / `Y` | 1, 2 | ダウンサンプル係数 |
| `NUM_SAMPLES_PER_PIXEL_1D` | 1/2/4/16 | サンプル数 |

---

## DenoiserTemporalCS 出力

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `RWDiffuseLighting` | `RWTexture2D<float3>` | テンポラル済み拡散光 |
| `RWSpecularLighting` | `RWTexture2D<float3>` | テンポラル済み鏡面光 |
| `RWLightingMoments` | `RWTexture2D<float4>` | 更新済み分散情報 |
| `RWNumFramesAccumulated` | `RWTexture2D<UNORM float>` | 蓄積フレーム数 |
| `RWShadingConfidence` | `RWTexture2D<UNORM float>` | シェーディング信頼度 |

---

## DenoiserSpatialCS パーミュテーション

| マクロ | 値 | 説明 |
|--------|-----|------|
| `INPUT_TYPE` | `INPUT_TYPE_GBUFFER` / `INPUT_TYPE_HAIRSTRANDS` | 入力ソース |
| `SPATIAL_FILTER` | 0/1 | 空間フィルタ有効/無効 |

---

## テンポラル蓄積アルゴリズム定数

| 定数 | 説明 |
|------|------|
| `MinFramesAccumulatedForHistoryMiss` | History ミス時の最小蓄積数（デフォルト 4）|
| `MinFramesAccumulatedForHighConfidence` | 高信頼度判定のしきい値 |
| `TemporalMaxFramesAccumulated` | 最大蓄積フレーム数（デフォルト 12）|
| `TemporalNeighborhoodClampScale` | Neighborhood Clamp スケール |

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `MegaLights.ush` | FMegaLightsData, IsLightingValid(), MegaLightsStateFrameIndex |
| `MegaLightsMaterial.ush` | LoadMaterial(), FMegaLightsMaterial, GetDenoisingModulateFactors() |
| `Lumen/LumenReflectionDenoiserCommon.ush` | リプロジェクション・Neighborhood Clamp 共通関数 |
| `StochasticLighting/StochasticLightingCommon.ush` | BRDF 評価補助関数 |
