# GPU d Ref: DF Shadows シェーダーリファレンス

- シェーダー: `DistanceFieldShadowing.usf`
- CPU 対応: [[ref_df_shadows]]
- 上位: [[01_distance_field_gpu_overview]]

---

## シェーダーファイル

| ファイル | エントリポイント | 役割 |
|---------|---------------|------|
| `DistanceFieldShadowing.usf` | `CullObjectsToShadowFrustumCS` | シャドウフラスタムカリング |
| `DistanceFieldShadowing.usf` | `RayTraceShadowsCS` | コーントレース（非タイル） |
| `DistanceFieldShadowing.usf` | `UpsamplePS` / Upsample系 | シャドウテクスチャのアップスケール |

---

## マクロ定義

| マクロ | 値 | 意味 |
|--------|-----|------|
| `CULLING_TYPE` | 0 | `SCATTER_TILE_CULLING`（タイルベース）|
| `CULLING_TYPE` | 2 | `POINT_LIGHT`（ポイント/スポットライト）|
| `DISTANCEFIELD_PRIMITIVE_TYPE` | `DFPT_SignedDistanceField` | SDF プリミティブ |
| `UPSAMPLE_PASS` | defined | アップスケールパス（別コンパイル）|

---

## グローバルパラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `ObjectBoundingGeometryIndexCount` | `uint` | IndirectDraw インデックス数 |
| `ObjectExpandScale` | `float` | オブジェクト影響範囲拡張スケール |
| `ShadowConvexHull[12]` | `float4[]` | シャドウフラスタム凸包（最大12面）|
| `ShadowBoundingSphere` | `float4` | バウンドスフィア (xyz=center, w=radius) |
| `NumShadowHullPlanes` | `uint` | 凸包の平面数 |
| `bDrawNaniteMeshes` | `uint` | Nanite メッシュ処理フラグ |
| `bCullHeighfieldsNotInAtlas` | `uint` | Atlas 未登録地形を除外するフラグ |

---

## ShadowConvexHullIntersectSphere

```hlsl
bool ShadowConvexHullIntersectSphere(float3 SphereOrigin, float SphereRadius)
```

シャドウフラスタム（凸包）とスフィアの交差判定。  
全平面に対してスフィアが外側にある場合は `false`（カリング）。

---

## ShadowConvexHullIntersectBox

```hlsl
bool ShadowConvexHullIntersectBox(float3 BoxOrigin, float3 BoxExtent)
```

シャドウフラスタムと AABB の交差判定。  
CullGrid の格子セル単位でのカリングに使用。

---

## コーントレース定数

| 定数 | 値 | 説明 |
|------|-----|------|
| `MAX_TRACE_SPHERE_RADIUS`（SDF）| 100 | SDF メッシュ用コーン球最大半径 |
| `MAX_TRACE_SPHERE_RADIUS`（HeightField）| 500 | 地形用コーン球最大半径 |
| `SELF_SHADOW_VERTICAL_BIAS`（SDF）| 0 | 地形のみバイアスあり |
| `SELF_SHADOW_VIEW_BIAS`（SDF）| 0 | 地形のみバイアスあり |

---

## FDistanceFieldShadowingParameters（CPU 側構築）

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `ObjectParameters` | `FDistanceFieldCulledObjectBufferParameters` | カリング済みオブジェクト SDF |
| `TanLightAngle` | `float` | ライト角度（コーン開口角）|
| `ShadowRayBias` | `float` | セルフシャドウ回避バイアス |
| `MaxOcclusionDistance` | `float` | コーントレース最大距離 |
| `bOnePassPointLightShadow` | `uint` | 一パスポイントライト |

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `DistanceFieldLightingShared.ush` | `LoadDFObjectData`, `LoadDFObjectBounds`, `DistanceToNearestSurfaceForObject` |
| `DistanceFieldShadowingShared.ush` | シャドウ専用型・`RayTraceShadow` ループ補助 |
| `MeshDistanceFieldCommon.ush` | Mesh SDF Atlas サンプリング |
| `Substrate/Substrate.ush` | Substrate マテリアルシステム対応 |

---

## 入出力テクスチャ

| バッファ/テクスチャ | 型 | 役割 |
|-----------------|-----|------|
| `SceneDepthTexture` | `Texture2D` | ピクセル深度（レイ起点計算）|
| `RWRayTracedShadowsTexture` | `RWTexture2D<float2>` | (Shadow, SSAO) 出力 |
| `RayTracedShadowsTexture` | `Texture2D<float2>` | アップスケール入力 |

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.DistanceFieldShadowing` | DF Shadows 有効/無効（0: 無効, 1: 有効）|
| `r.DFShadowQuality` | 品質レベル（ステップ数・コーン精度）|
| `r.DistanceFieldShadowing.UseAsyncCompute` | 非同期 Compute 使用 |
| `r.DistanceFieldShadowing.MaxObjectsPerShadow` | ライトあたり最大オブジェクト数 |
| `r.DistanceFieldShadowing.RayBias` | セルフシャドウ回避バイアス |
