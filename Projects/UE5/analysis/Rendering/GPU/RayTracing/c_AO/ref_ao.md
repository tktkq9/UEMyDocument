# GPU c Ref: AO シェーダーリファレンス

- シェーダー: `RayTracing/RayTracingAmbientOcclusionRGS.usf`, `RayTracingSkyLightRGS.usf`, `GenerateSkyLightVisibilityRaysCS.usf`
- CPU 対応: [[b_rt_shadow_ao]]
- 上位: [[01_raytracing_gpu_overview]]

---

## エントリポイント一覧

### RayTracingAmbientOcclusionRGS.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `AmbientOcclusionRGS` | 85 | RGS | ヘミスフェアレイで AO 計算 |

### RayTracingSkyLightRGS.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `SkyLightRGS` | 32 | RGS | 天空方向レイで SkyLight 遮蔽評価 |

### GenerateSkyLightVisibilityRaysCS.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `MainCS` | 18 | CS | SkyLight 可視性レイ方向の事前生成 |

### 合成パス

| ファイル | タイプ | 説明 |
|---------|--------|------|
| `CompositeAmbientOcclusionPS.usf` | PS | AO マスクを SceneColor に合成 |
| `CompositeSkyLightPS.usf` | PS | Sky 遮蔽マスクを SceneColor に合成 |

---

## AmbientOcclusionRGS パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `TLAS` | `RaytracingAccelerationStructure` | シーン TLAS |
| `SamplesPerPixel` | `uint` | ピクセルあたりサンプル数 |
| `MaxRayDistance` | `float` | 最大 AO 距離 |
| `Intensity` | `float` | AO 強度スケール |
| `MaxNormalBias` | `float` | 法線バイアス上限 |
| `RWAmbientOcclusionMaskUAV` | `RWTexture2D<float>` | 出力: AO マスク |
| `RWAmbientOcclusionHitDistanceUAV` | `RWTexture2D<float>` | 出力: ヒット距離 |

---

## SkyLightRGS パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `TLAS` | `RaytracingAccelerationStructure` | シーン TLAS |
| `UpscaleFactor` | `uint` | アップスケール係数 |
| `RWSkyOcclusionMaskUAV` | `RWTexture2D<float4>` | 出力: Sky 遮蔽マスク |
| `RWSkyOcclusionRayDistanceUAV` | `RWTexture2D<float2>` | 出力: ヒット距離 |

---

## パーミュテーション

### AmbientOcclusionRGS

| マクロ | 値 | 説明 |
|--------|-----|------|
| `SUBSTRATE_GBUFFER_FORMAT` | 0/1 | Substrate GBuffer 形式 |
| `CONFIG_SHOOT_WITH_GEOMETRIC_NORMAL` | 1 | 幾何法線でサンプリング（デフォルト）|

### SkyLightRGS

| マクロ | 値 | 説明 |
|--------|-----|------|
| `USE_HAIR_LIGHTING` | 0/1 | Hair Strands ライティング |
| `SUBSTRATE_ENABLED` | 0/1 | Substrate マテリアル対応 |
| `SUBSTRATE_GBUFFER_FORMAT` | 0/1 | Substrate GBuffer 形式 |

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `RayTracingSkyLightCommon.ush` | SkyLight 共通型・定数 |
| `RayTracingSkyLightEvaluation.ush` | SampleSkyLightVisibility(), EvalSkyLight() |
| `SkyLightVisibilityRaysData.ush` | 可視性レイデータ型 |
| `MipTreeCommon.ush` / `SkyLightMipTreeCommon.ush` | 重要度サンプリング用 MipTree |
| `PathTracing/Utilities/PathTracingRandomSequence.ush` | ランダムシーケンス |
