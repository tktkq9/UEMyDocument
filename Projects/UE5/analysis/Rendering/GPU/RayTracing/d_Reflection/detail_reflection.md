# GPU d: Reflection — HW Ray Traced Reflections

- シェーダー: `RayTracing/RayTracingPrimaryRays.usf`, `RayTracingMaterialHitShaders.usf`, `RayTracingDeferredMaterials.usf`
- CPU 対応: [[c_rt_reflection]]
- 上位: [[01_raytracing_gpu_overview]]

---

## 概要

カメラから GBuffer 上の各ピクセルへ1次レイを発射し、  
ヒット先でフルマテリアル評価（GGX + Lambertian + ライティング）を行う高品質リフレクション。  
`RayTracingPrimaryRaysRGS` が反射・透過の両方を処理するエントリポイントとなる。

---

## RayTracingPrimaryRaysRGS（RayTracingPrimaryRays.usf: line 89）

```hlsl
RAY_TRACING_ENTRY_RAYGEN(RayTracingPrimaryRaysRGS)
{
    uint2 PixelCoord = GetPixelCoord(DispatchRaysIndex().xy, UpscaleFactor);

    // 1. GBuffer の深度から TranslatedWorldPosition を再構築
    // 2. CreatePrimaryRay() でカメラレイを生成
    //    [ERayTracingPrimaryRaysFlag_UseGBufferForMaxDistance]
    //    TMax = GBuffer 深度を上限に設定（半透明オブジェクトのみトレース）
    // 3. FRayCone を初期化（RayCone.SpreadAngle = EyeToPixelSpreadAngle）
    // 4. TraceRayAndAccumulateResults() で1次ヒットを計算
    //    → HitGroup: MaterialCHS / MaterialAHS
    //    → Miss: RayTracingLightingMS（環境光評価）
    // 5. [Reflection 有効]
    //    Payload からラフネス・法線を取得
    //    ImportanceSampleGGX() で反射方向を生成
    //    TraceRayAndAccumulateResults() で2次レイ（反射）を発射
    //    ReflectionPayload.IsMiss() → Sky 方向の評価
    // 6. RWColorOutput[PixelCoord] に最終色を書き込み
}
```

**パーミュテーション:**

| マクロ | 値 | 説明 |
|--------|-----|------|
| `ERayTracingPrimaryRaysFlag_UseGBufferForMaxDistance` | ビット | GBuffer 深度で TMax を制限 |
| `ENABLE_TWO_SIDED_GEOMETRY` | 0/1 | 両面ジオメトリ対応 |
| `SUBSTRATE_ENABLED` | 0/1 | Substrate マテリアル対応 |

---

## MaterialCHS（RayTracingMaterialHitShaders.usf: line 630）

マテリアルの完全評価を行う Closest Hit Shader。  
フルシェーディング（テクスチャサンプリング・ライティング）を実行する。

```hlsl
RAY_TRACING_ENTRY_CLOSEST_HIT(MaterialCHS,
    FPackedMaterialClosestHitPayload, PackedPayload,
    FRayTracingIntersectionAttributes, Attributes)
{
    // 1. RayTracingCalcInterpolants.ush: CalcInterpolants() で UV・法線を補間
    // 2. マテリアルシェーダーを評価（CalcMaterialParameters → GetMaterialPayload）
    //    → Emissive, BaseColor, Metallic, Roughness, Normal を計算
    // 3. GetMaterialPayload() で FMaterialClosestHitPayload に格納
    //    - Substrate 有効時: SubstrateTree を評価
    //    - 従来 GBuffer 時: Standard BRDF パラメータを格納
    // 4. PackedPayload に詰めて返す
}
```

---

## MaterialAHS（RayTracingMaterialHitShaders.usf: line 794）

アルファテストや Masked マテリアルのための Any Hit Shader。

```hlsl
RAY_TRACING_ENTRY_ANY_HIT(MaterialAHS,
    FPackedMaterialClosestHitPayload, PackedPayload,
    FRayTracingIntersectionAttributes, Attributes)
{
    // GetMaterialOutputOpacity() でアルファを計算
    // Opacity < 0.5 → IgnoreHit()（透過）
    // Opacity >= 0.5 → AcceptHitAndEndSearch()（遮蔽）
}
```

---

## RayTracingDeferredMaterials.usf

ヒットシェーダーの評価を後回しにして Deferred 方式でマテリアルを評価するシェーダー群。  
ヒット時はライト ID のみ保存し、後続 CS でまとめて評価することでシェーダー切り替えコストを削減。

```hlsl
// 主な役割: RayTracingGatherPoints + MaterialShading の分離
// → RayTracingGatherPoints.ush と組み合わせて使用
```

---

## レイコーン（RayCone）とテクスチャ LOD

```hlsl
// ヒット距離に応じてコーンが広がり、テクスチャ LOD が自動計算される
// FRayCone.SpreadAngle = EyeToPixelSpreadAngle（カメラの視野角/解像度）
// ヒット時: ConeDiameter = SpreadAngle × RayDistance
//         → ComputeRayConeLOD() で MIP レベルを決定
```

---

## リフレクション パイプライン

```
[CPU] RenderRayTracingReflections()
  ↓
RayTracingPrimaryRaysRGS
  → 1次レイ → MaterialCHS（マテリアル評価）
  → 2次レイ（反射方向）→ MaterialCHS / RayTracingLightingMS
  ↓
RWColorOutput（ノイジーなリフレクション結果）
  ↓
IScreenSpaceDenoiser::DenoiseReflections()
  ↓
最終 SceneColor に合成
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RayTracing.Reflections` | -1 | -1=自動 / 0=無効 / 1=有効 |
| `r.RayTracing.Reflections.SamplesPerPixel` | 1 | サンプル数 |
| `r.RayTracing.Reflections.MaxRayDistance` | -1 | 最大反射距離 |
| `r.RayTracing.Reflections.MaxRoughness` | 0.6 | 反射を適用する最大ラフネス |
