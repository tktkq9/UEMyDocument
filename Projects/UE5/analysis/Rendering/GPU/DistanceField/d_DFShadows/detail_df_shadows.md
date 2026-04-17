# GPU d: DF Shadows — Mesh SDF コーントレースシャドウシェーダー

- シェーダー: `DistanceFieldShadowing.usf`
- CPU 対応: [[d_df_shadows]] | [[ref_df_shadows]]
- 上位: [[01_distance_field_gpu_overview]]

---

## 概要

DF Shadows GPU パスは `DistanceFieldShadowing.usf` の Compute Shader で処理される。  
Object SDF バッファから影響オブジェクトをカリングし、Mesh SDF Atlas をサンプリングして  
コーントレースによるソフトシャドウを計算する。

---

## シェーダーのパーミュテーション

`DistanceFieldShadowing.usf` では `CULLING_TYPE` マクロでパスを切り替える。

| `CULLING_TYPE` | マクロ定義 | 用途 |
|-------------|-----------|------|
| 0 | `SCATTER_TILE_CULLING` | タイルベースカリング（DirectionalLight 等）|
| 2 | `POINT_LIGHT` | ポイント/スポットライト |

**`DISTANCEFIELD_PRIMITIVE_TYPE` による分岐：**

| 定義値 | `MAX_TRACE_SPHERE_RADIUS` | バイアス値 |
|------|--------------------------|---------|
| `DFPT_SignedDistanceField` | 100 | 0（SDF は精度が高い）|
| その他（HeightField 等）| 500 | あり（SELF_SHADOW_VERTICAL_BIAS = 100）|

---

## CS パス構成

```
1. CullObjectsToShadowFrustumCS
   → シャドウフラスタム + ConvexHull 交差で SDF オブジェクトをカリング

2. RayTraceShadowsCS（非 Tiled）または ConeTraceShadowsCS（Tiled）
   → オブジェクト SDF + Mesh SDF Atlas でコーントレース
   → RayTracedShadowsTexture へ遮蔽率を出力
```

---

## CullObjectsToShadowFrustumCS

```hlsl
bool ShadowConvexHullIntersectSphere(float3 SphereOrigin, float SphereRadius)
{
    for (uint PlaneIndex = 0; PlaneIndex < NumShadowHullPlanes; PlaneIndex++)
    {
        float4 PlaneData = ShadowConvexHull[PlaneIndex];
        float PlaneDistance = dot(PlaneData.xyz, SphereOrigin) - PlaneData.w;
        if (PlaneDistance > SphereRadius) return false;
    }
    return true;
}
```

- `ShadowConvexHull[12]`（最大 12 平面）でシャドウ投影ボリュームを定義
- `ShadowBoundingSphere` との二段階チェックで高速カリング
- `bCullHeighfieldsNotInAtlas` フラグで地形 SDF を選択的に除外

---

## コーントレース処理

```hlsl
// 擬似コード: RayTraceShadowsCS のコーントレースループ
float RayTraceShadow(float3 RayStart, float3 RayDir, float TanLightAngle, float MaxDist)
{
    float MinShadowPenumbra = 1.0f;
    float StepDist = ShadowRayStartOffset;  // セルフシャドウ回避のオフセット

    while (StepDist < MaxDist)
    {
        float3 WorldPos = RayStart + RayDir * StepDist;

        // 影響オブジェクトを走査
        for each CulledObjectIndex:
        {
            // DistanceToNearestSurfaceForObject で Mesh SDF をサンプリング
            float Distance = DistanceToNearestSurfaceForObject(ObjectData, WorldPos, ...);

            // コーン開口角（光源サイズ）に対してペナンブラを計算
            float SphereRadius = StepDist * TanLightAngle;
            float Penumbra = saturate(Distance / SphereRadius);
            MinShadowPenumbra = min(MinShadowPenumbra, Penumbra);
        }

        // ステップサイズを更新（SDF 値を参考に効率的なステップ）
        StepDist += max(MinDistance * StepScale, MIN_STEP_SIZE);
    }

    return MinShadowPenumbra; // 1.0 = 完全明るい, 0.0 = 完全影
}
```

---

## ソフトシャドウの原理

```
光源サイズ → コーン角（TanLightAngle）
  │
  ↓ コーン開口角が大きいほど、遮蔽物が小さい or 遠い場合に半影が広がる

TanLightAngle = LightSourceRadius / DistanceToLight

ペナンブラ係数 = SDF距離 / (StepDist * TanLightAngle)
  → 1 に近い = 影の外（明るい）
  → 0 に近い = 影の内側（暗い）
  → 中間値 = 半影（ソフトシャドウ）
```

---

## ObjectExpandScale（メッシュ拡張）

```cpp
float ObjectExpandScale; // シェーダーパラメータ
```

Nanite メッシュや小さなオブジェクトの SDF が不正確な場合に  
オブジェクトの影響範囲をわずかに拡張してアーティファクトを低減する。

---

## bDrawNaniteMeshes フラグ

```cpp
uint bDrawNaniteMeshes; // シェーダーパラメータ
```

Nanite メッシュの DF Shadows 処理を有効化するフラグ。  
Nanite は通常の SDF 生成パイプラインとは異なる SDF 管理を持つ。

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `DistanceFieldLightingShared.ush` | `LoadDFObjectData`, `LoadDFObjectBounds`, Atlas サンプリング |
| `DistanceFieldShadowingShared.ush` | シャドウ専用共有型・関数 |
| `MeshDistanceFieldCommon.ush` | `DistanceToNearestSurfaceForObject` |

---

## 入出力バッファ

| バッファ | 型 | 方向 | 説明 |
|--------|-----|------|------|
| `SceneObjectData` | `StructuredBuffer<FDFObjectData>` | 読み取り | Object SDF データ全体 |
| `SceneObjectBounds` | `StructuredBuffer<FDFObjectBounds>` | 読み取り | Object バウンド |
| `RWRayTracedShadowsTexture` | `RWTexture2D<float2>` | 書き込み | 遮蔽率 出力テクスチャ |
| `SceneDepthTexture` | `Texture2D` | 読み取り | 深度（レイ起点計算）|
