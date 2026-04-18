# GPU e Ref: Translucency シェーダーリファレンス

- シェーダー: `RayTracing/RayTracingPrimaryRays.usf`, `RayTracingMaterialHitShaders.usf`, `RayTracingDeferredMaterials.usf`, `RayTracingLightingMS.usf`
- CPU 対応: [[c_rt_reflection]]
- 上位: [[01_raytracing_gpu_overview]]

---

## エントリポイント一覧

### RayTracingPrimaryRays.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `RayTracingPrimaryRaysRGS` | 89 | RGS | 半透明・屈折レイ発射（Translucent パーミュテーション）|

### RayTracingMaterialHitShaders.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `MaterialCHS` | 630 | CHS | 半透明マテリアル Closest Hit（Opacity + Radiance を返す）|
| `MaterialAHS` | 794 | AHS | Masked マテリアル Any Hit（IgnoreHit / AcceptHitAndEndSearch）|

### RayTracingLightingMS.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `RayTracingLightingMS` | 55 | MS | 半透明レイの Miss 時に環境光を評価 |

---

## RayTracingPrimaryRaysRGS パラメータ（半透明モード）

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `TLAS` | `RaytracingAccelerationStructure` | シーン TLAS |
| `PrimaryRayFlags` | `uint` | `ERayTracingPrimaryRaysFlag_*` ビットフラグ |
| `MaxRefractionRays` | `uint` | 最大屈折レイ数（デフォルト 3）|
| `SamplesPerPixel` | `uint` | ピクセルあたりサンプル数 |
| `MaxRayDistance` | `float` | 最大距離 |
| `RWColorOutput` | `RWTexture2D<float4>` | 出力: 半透明合成色 |

---

## レイマスク

| 定数 | 説明 |
|------|------|
| `RAY_TRACING_MASK_TRANSLUCENT` | 半透明ジオメトリのみ |
| `RAY_TRACING_MASK_OPAQUE` | 不透明ジオメトリのみ |
| `RAY_TRACING_MASK_ALL` | 全ジオメトリ |

---

## FMaterialClosestHitPayload — 半透明関連フィールド

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `Radiance` | `float3` | ライティング評価済み輝度 |
| `Opacity` | `float` | 不透明度（0=完全透明 / 1=完全不透明）|
| `TransparencyColor` | `float3` | 透過色（Absorptive 半透明）|
| `IOR` | `float` | 屈折率（屈折計算に使用）|
| `IsMiss()` | `bool` | Miss Shader から返ってきた場合 true |

---

## パーミュテーション

### RayTracingPrimaryRaysRGS

| マクロ | 値 | 説明 |
|--------|-----|------|
| `SUBSTRATE_ENABLED` | 0/1 | Substrate マテリアル対応 |
| `ENABLE_TWO_SIDED_GEOMETRY` | 0/1 | 両面ジオメトリ |

### MaterialAHS

| マクロ | 値 | 説明 |
|--------|-----|------|
| `SUBSTRATE_ENABLED` | 0/1 | Substrate マテリアル対応 |
| `SUBSTRATE_GBUFFER_FORMAT` | 0/1 | Substrate GBuffer 形式 |

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `RayTracingCommon.ush` | ペイロード型・TraceRay ラッパー・IsMiss()/IsHit() |
| `RayTracingDeferredShadingCommon.ush` | GBuffer 参照・ライティング評価 |
| `RayTracingHitGroupCommon.ush` | ヒットグループ共通定義 |
| `RayTracingCalcInterpolants.ush` | UV・法線・タンジェント補間 |
| `RayTracingDeferredMaterials.ush` | 遅延マテリアル評価型定義 |
