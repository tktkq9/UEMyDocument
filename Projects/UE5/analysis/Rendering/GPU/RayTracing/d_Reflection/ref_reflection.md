# GPU d Ref: Reflection シェーダーリファレンス

- シェーダー: `RayTracing/RayTracingPrimaryRays.usf`, `RayTracingMaterialHitShaders.usf`, `RayTracingDeferredMaterials.usf`
- CPU 対応: [[c_rt_reflection]]
- 上位: [[01_raytracing_gpu_overview]]

---

## エントリポイント一覧

### RayTracingPrimaryRays.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `RayTracingPrimaryRaysRGS` | 89 | RGS | 1次・2次レイ発射（反射・透過）|

### RayTracingMaterialHitShaders.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `MaterialCHS` | 630 | CHS | フルマテリアル評価 Closest Hit |
| `MaterialAHS` | 794 | AHS | アルファテスト Any Hit |

### RayTracingLightingMS.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `RayTracingLightingMS` | 55 | MS | 環境光評価 Miss Shader |

---

## RayTracingPrimaryRaysRGS パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `TLAS` | `RaytracingAccelerationStructure` | シーン TLAS |
| `UpscaleFactor` | `uint` | アップスケール係数 |
| `PrimaryRayFlags` | `uint` | `ERayTracingPrimaryRaysFlag_*` ビットフラグ |
| `MaxRefractionRays` | `uint` | 最大屈折レイ数 |
| `SamplesPerPixel` | `uint` | ピクセルあたりサンプル数 |
| `MaxRayDistance` | `float` | 最大レイ距離 |
| `SkyAverageBrightness` | `float` | スカイ平均輝度（ミス時）|
| `RWColorOutput` | `RWTexture2D<float4>` | 出力: リフレクション結果 |

---

## ERayTracingPrimaryRaysFlag

| フラグ | 説明 |
|--------|------|
| `UseGBufferForMaxDistance` | GBuffer 深度で TMax を制限 |
| `AllowSkySampling` | Miss 時にスカイをサンプリング |
| `PrimaryView` | カメラ視点の1次レイ |

---

## パーミュテーション（MaterialCHS）

| マクロ | 値 | 説明 |
|--------|-----|------|
| `SUBSTRATE_ENABLED` | 0/1 | Substrate マテリアル対応 |
| `SUBSTRATE_GBUFFER_FORMAT` | 0/1 | Substrate GBuffer 形式 |
| `ENABLE_TWO_SIDED_GEOMETRY` | 0/1 | 両面ジオメトリ |
| `USE_RAYTRACED_TEXTURE_RAYCONE_LOD` | 0/1 | レイコーン LOD |

---

## ペイロード型詳細

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `FMaterialClosestHitPayload.Radiance` | `float3` | Emissive 輝度 |
| `FMaterialClosestHitPayload.BaseColor` | `float3` | ベースカラー |
| `FMaterialClosestHitPayload.Normal` | `float3` | シェーディング法線 |
| `FMaterialClosestHitPayload.Metallic` | `float` | メタリック |
| `FMaterialClosestHitPayload.Roughness` | `float` | ラフネス |
| `FMaterialClosestHitPayload.Opacity` | `float` | 不透明度 |
| `IsMiss()` | `bool` | Miss シェーダーから返った場合 true |
| `IsHit()` | `bool` | ヒットした場合 true |

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `RayTracingCommon.ush` | ペイロード型・TraceRay ラッパー |
| `RayTracingDeferredShadingCommon.ush` | GBuffer 参照・ライティング評価 |
| `RayTracingCalcInterpolants.ush` | UV・法線補間 |
| `RayTracingHitGroupCommon.ush` | ヒットグループ共通定義 |
| `RayTracingReflectionEnvironment.ush` | 反射環境キャプチャ評価 |
| `RayTracingRayCone.ush` | レイコーン LOD 計算 |
