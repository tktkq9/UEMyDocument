# GPU RayTracing シェーダー全体概要

- CPU 対応: [[07_raytracing_overview]]
- 上位: [[01_gpu_overview]]

---

## RayTracing GPU パイプライン

`RHI_RAYTRACING` が有効な場合のみコンパイルされる HW アクセラレーション RT 基盤。  
DXR (DirectX 12) / Vulkan Ray Tracing 対応 GPU でのみ動作する。

```
【A】 TLAS/BLAS 構築
  RayTracingInstanceBufferUtil.usf: BuildRayTracingInstanceBufferCS
  RayTracingDynamicMesh.usf: RayTracingDynamicGeometryConverterCS
  → BLAS インスタンスを GPU バッファに書き込み
  → RHI が TLAS（Top Level Acceleration Structure）をビルド

【B】 シャドウ（Occlusion）
  RayTracingOcclusionRGS.usf: Occlusion（RGS / Inline CS）
  → 各ライト（Directional/Spot/Point/Rect）の影レイを発射
  → OcclusionMask テクスチャに可視性を書き込み
  → IScreenSpaceDenoiser でデノイズ

【C】 アンビエントオクルージョン（AO）
  RayTracingAmbientOcclusionRGS.usf: AmbientOcclusionRGS
  → コサインサンプリングでヘミスフェアレイを発射
  → AmbientOcclusionMask に書き込み

【D】 スカイライト（SkyLight）
  RayTracingSkyLightRGS.usf: SkyLightRGS
  GenerateSkyLightVisibilityRaysCS.usf: MainCS
  → 天空方向へのレイで sky occlusion を計算

【E】 リフレクション（Primary Rays）
  RayTracingPrimaryRays.usf: RayTracingPrimaryRaysRGS
  RayTracingMaterialHitShaders.usf: MaterialCHS / MaterialAHS
  → 反射・屈折レイを発射し Material Shading を実行

【F】 半透明（Translucency）
  RayTracingPrimaryRays.usf: RayTracingPrimaryRaysRGS（Translucent パーミュテーション）
  RayTracingLightingMS.usf: RayTracingLightingMS
  → 半透明オブジェクトにレイを発射しライティングを評価

【G】 組み込みシェーダー（Built-in）
  RayTracingBuiltInShaders.usf: OcclusionMainRGS / IntersectionMainRGS / DefaultMainCHS 等
  → テスト・デバッグ・デフォルトフォールバック用シェーダー
```

---

## シェーダーグループ一覧

| グループ | 主なファイル | 役割 |
|--------|-----------|------|
| **a: Core** | `RayTracingBuiltInShaders.usf`, `RayTracingInstanceBufferUtil.usf`, `RayTracingDynamicMesh.usf` | TLAS/BLAS 構築・組み込みシェーダー |
| **b: Shadow** | `RayTracingOcclusionRGS.usf`, `RayTracingMaterialDefaultHitShaders.usf` | HW RT シャドウ（Occlusion レイ）|
| **c: AO** | `RayTracingAmbientOcclusionRGS.usf`, `RayTracingSkyLightRGS.usf`, `GenerateSkyLightVisibilityRaysCS.usf` | RT AO + RT スカイライト |
| **d: Reflection** | `RayTracingPrimaryRays.usf`, `RayTracingMaterialHitShaders.usf`, `RayTracingLightingMS.usf` | RT リフレクション・マテリアルヒットシェーダー |
| **e: Translucency** | `RayTracingPrimaryRays.usf`（Translucent パス）, `RayTracingDeferredMaterials.usf` | RT 半透明・遅延マテリアル評価 |

---

## シェーダーエントリタイプ

| タイプ | マクロ | 説明 |
|--------|--------|------|
| Ray Generation | `RAY_TRACING_ENTRY_RAYGEN` | レイ発射の起点（1ピクセル = 1スレッド）|
| Closest Hit | `RAY_TRACING_ENTRY_CLOSEST_HIT` | 最近傍ヒット時に実行（マテリアル評価）|
| Any Hit | `RAY_TRACING_ENTRY_ANY_HIT` | 任意ヒット時（アルファテスト等）|
| Miss | `RAY_TRACING_ENTRY_MISS` | レイがどこにもヒットしない場合 |
| Inline（CS）| `RAY_TRACING_ENTRY_RAYGEN_OR_INLINE` | CS 内で TraceRayInline() 使用 |

---

## 主要ペイロード型

| 型 | ファイル | 用途 |
|-----|--------|------|
| `FDefaultPayload` | `RayTracingCommon.ush` | 最小ペイロード（Occlusion 用）|
| `FPackedMaterialClosestHitPayload` | `RayTracingCommon.ush` | マテリアル評価結果（圧縮済み）|
| `FMaterialClosestHitPayload` | `RayTracingCommon.ush` | マテリアル評価結果（展開済み）|

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `RayTracingCommon.ush` | ペイロード型定義・共通関数 |
| `RayTracingDeferredShadingCommon.ush` | GBuffer 参照・遅延シェーディング補助 |
| `RayTracingHitGroupCommon.ush` | ヒットグループ共通処理 |
| `RayTracingCalcInterpolants.ush` | 補間データ計算（UV・法線補間）|
| `TraceRayInline.ush` | Inline Ray Tracing（CS内 TraceRayInline）|
| `RayGenUtils.ush` | RayGen 共通ユーティリティ |
