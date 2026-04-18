# GPU e: Translucency — Ray Traced Translucency

- シェーダー: `RayTracing/RayTracingPrimaryRays.usf`（Translucent パス）, `RayTracingMaterialHitShaders.usf`, `RayTracingLightingMS.usf`, `RayTracingDeferredMaterials.usf`
- CPU 対応: [[c_rt_reflection]]
- 上位: [[01_raytracing_gpu_overview]]

---

## 概要

半透明オブジェクトに対してレイトレーシングを行い、  
深度ソートなしで正確な屈折・ライティングを計算する高品質モード。  
`RayTracingPrimaryRaysRGS` が半透明・反射の両方に対応しており、  
パーミュテーション（`ERayTracingPrimaryRaysFlag`）で動作を切り替える。

---

## RayTracingPrimaryRaysRGS — 半透明パス（RayTracingPrimaryRays.usf: line 89）

```hlsl
RAY_TRACING_ENTRY_RAYGEN(RayTracingPrimaryRaysRGS)
{
    // [半透明モード]
    // UseGBufferForMaxDistance フラグ有効:
    //   TMax = GBuffer 上の不透明オブジェクト深度
    //   → 不透明オブジェクトより手前の半透明だけをトレース

    // 1. RAY_TRACING_MASK_TRANSLUCENT でレイを発射
    //    → 半透明オブジェクトの MaterialCHS を実行
    // 2. Payload.Opacity を使って前から後ろへアルファブレンド
    //    AccumColor = lerp(AccumColor, Payload.Radiance, Payload.Opacity)
    // 3. MaxRefractionRays 回の屈折レイを再帰発射
    //    → IOR（屈折率）から Snell の法則で次のレイ方向を計算
    // 4. ヒットしなかった場合（IsMiss）:
    //    RayTracingLightingMS → Sky や環境光を評価
    // 5. RWColorOutput[PixelCoord] = 最終合成色
}
```

**半透明フラグ:**

| フラグ | 説明 |
|--------|------|
| `ERayTracingPrimaryRaysFlag_UseGBufferForMaxDistance` | 不透明より手前の半透明のみをトレース |
| `ERayTracingPrimaryRaysFlag_AllowSkySampling` | Miss 時にスカイをサンプリング |

---

## 屈折処理

```hlsl
// MaterialCHS 内で屈折方向を計算
float3 RefractedDirection = refract(RayDirection, ShadingNormal, IOR_Ratio);
// IOR_Ratio = IOR_Air / IOR_Material（空気→マテリアル方向の場合）

// 屈折レイは次のイテレーションで再発射
// MaxRefractionRays（デフォルト 3）を超えた場合は中止
```

---

## MaterialCHS（RayTracingMaterialHitShaders.usf: line 630）— 半透明での動作

半透明マテリアルの場合、ClosestHit でフルシェーディングを実行し  
Opacity と Radiance の両方を `FPackedMaterialClosestHitPayload` に格納して返す。

```hlsl
RAY_TRACING_ENTRY_CLOSEST_HIT(MaterialCHS, ...)
{
    // [半透明マテリアル]
    // GetMaterialOutputOpacity() で Opacity を計算
    // Emissive + ライティング評価結果を Radiance に格納
    // → Payload.Opacity, Payload.Radiance をセット
    //   （呼び出し元でアルファブレンドに使用）
}
```

---

## MaterialAHS（RayTracingMaterialHitShaders.usf: line 794）— 半透明での動作

Masked マテリアル（完全透明/不透明のみ）用 Any Hit Shader。  
半透明グラデーションのある場合は CHS 側で処理する。

```hlsl
RAY_TRACING_ENTRY_ANY_HIT(MaterialAHS, ...)
{
    float Opacity = GetMaterialOutputOpacity();
    if (Opacity < OPACITY_THRESHOLD)
        IgnoreHit();       // 透過: このヒットを無視して続けてトレース
    else
        AcceptHitAndEndSearch();  // 不透明: 即決定
}
```

---

## RayTracingDeferredMaterials.usf — 遅延マテリアル評価

ヒット時にマテリアル評価を遅らせ、後続 CS でまとめて実行する最適化。

```hlsl
// ヒット時: GBuffer 座標と HitGroup ID のみを書き込む
// 後続: FRayTracingDeferredMaterialsCS で一括マテリアル評価
// → 同一マテリアルのヒットをまとめることでシェーダー切り替えを削減
```

---

## RayTracingLightingMS（半透明での役割）

半透明レイが背景（スカイ等）にヒットしなかった場合の Miss Shader。  
Reflection Capture や Sky Light からの環境光を評価する。

```hlsl
RAY_TRACING_ENTRY_MISS(RayTracingLightingMS, FPackedMaterialClosestHitPayload, PackedPayload)
{
    // GetSkyLightReflection() でスカイライト方向の輝度を取得
    // → PackedPayload.Radiance に書き込み
}
```

---

## 半透明パイプライン

```
[CPU] RenderRayTracingTranslucency()
  ↓
RayTracingPrimaryRaysRGS（RAY_TRACING_MASK_TRANSLUCENT）
  → 1次レイ → MaterialCHS（Opacity + Radiance 取得）
  → 屈折レイ（MaxRefractionRays 回）→ MaterialCHS / RayTracingLightingMS
  ↓
RWColorOutput（半透明合成済み色）
  ↓
SceneColor にアルファブレンドで合成
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RayTracing.Translucency` | -1 | -1=自動 / 0=無効 / 1=有効 |
| `r.RayTracing.Translucency.SamplesPerPixel` | 1 | サンプル数 |
| `r.RayTracing.Translucency.MaxRayDistance` | -1 | 最大距離 |
| `r.RayTracing.Translucency.MaxRefractionRays` | 3 | 最大屈折レイ数 |
| `r.RayTracing.Translucency.MaxRoughness` | 0.6 | 半透明に適用する最大ラフネス |
