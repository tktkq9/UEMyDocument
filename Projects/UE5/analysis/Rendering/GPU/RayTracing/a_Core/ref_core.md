# GPU a Ref: Core シェーダーリファレンス

- シェーダー: `RayTracing/RayTracingBuiltInShaders.usf`, `RayTracingInstanceBufferUtil.usf`, `RayTracingDynamicMesh.usf`, `RayTracingLightingMS.usf`
- CPU 対応: [[a_rt_scene]]
- 上位: [[01_raytracing_gpu_overview]]

---

## エントリポイント一覧

### RayTracingBuiltInShaders.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `OcclusionMainRGS` | 12 | RGS | TLAS 接続テスト用 Ray Generation |
| `IntersectionMainRGS` | 47 | RGS | 交差テスト用 Ray Generation |
| `IntersectionMainCHS` | 76 | CHS | 交差テスト用 Closest Hit |
| `DefaultMainCHS` | 86 | CHS | デフォルト Closest Hit（シェーディングなし）|
| `DefaultOpaqueAHS` | 97 | AHS | デフォルト Any Hit（不透明・即 Accept）|
| `DefaultPayloadMS` | 103 | MS | `FDefaultPayload` 用 Miss |
| `PackedMaterialClosestHitPayloadMS` | 108 | MS | `FPackedMaterialClosestHitPayload` 用 Miss |

### RayTracingInstanceBufferUtil.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `BuildRayTracingInstanceBufferCS` | 226 | CS | インスタンスバッファ構築 |

### RayTracingDynamicMesh.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `RayTracingDynamicGeometryConverterCS` | 45 | CS | 動的ジオメトリ頂点変換 |

### RayTracingLightingMS.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `RayTracingLightingMS` | 55 | MS | 環境ライティング評価 Miss Shader |

---

## BuildRayTracingInstanceBufferCS パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `InstanceInputBuffer` | `StructuredBuffer<FRayTracingInstanceDescInput>` | CPU 側インスタンス情報 |
| `InstanceOutputBuffer` | `RWStructuredBuffer<uint4>` | D3D12 インスタンス記述子バッファ |
| `NumInstances` | `uint` | インスタンス総数 |
| `bApplyLocalBoundsTransform` | `uint` | ローカル境界変換の適用 |

---

## ペイロード型

| 型 | 用途 |
|-----|------|
| `FDefaultPayload` | Hit/Miss 判定のみ（最小コスト）|
| `FPackedMaterialClosestHitPayload` | マテリアル評価結果（圧縮形式・64 bytes）|
| `FMaterialClosestHitPayload` | マテリアル評価結果（展開形式）|
| `FIntersectionPayload` | ヒット座標・法線（交差テスト用）|

---

## インスタンスマスク定数（RayTracingDefinitions.h）

| 定数 | 説明 |
|------|------|
| `RAY_TRACING_MASK_OPAQUE` | 不透明ジオメトリ |
| `RAY_TRACING_MASK_TRANSLUCENT` | 半透明ジオメトリ |
| `RAY_TRACING_MASK_SHADOW` | シャドウキャスター |
| `RAY_TRACING_MASK_ALL` | 全ジオメトリ |

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `RayTracingCommon.ush` | ペイロード型・TraceOcclusionRay()・TraceRay ラッパー |
| `RayTracingHitGroupCommon.ush` | ヒットグループ共通定義 |
| `TraceRayInline.ush` | Compute Shader 内 Inline Ray Tracing |
| `RayGenUtils.ush` | RayGen 補助関数（DispatchRaysIndex 等）|
