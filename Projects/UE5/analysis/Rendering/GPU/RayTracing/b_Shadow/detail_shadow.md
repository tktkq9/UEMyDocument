# GPU b: Shadow — HW Ray Traced Shadow

- シェーダー: `RayTracing/RayTracingOcclusionRGS.usf`, `RayTracingMaterialDefaultHitShaders.usf`
- CPU 対応: [[b_rt_shadow_ao]]
- 上位: [[01_raytracing_gpu_overview]]

---

## 概要

Directional / Spot / Point / Rect ライット向けのハードウェアレイトレーシングシャドウ。  
`RAY_TRACING_ENTRY_RAYGEN_OR_INLINE(Occlusion)` によって  
**RayGen シェーダー**（フル HW RT）と **Compute Shader 内 Inline Ray Tracing** の  
2 つのバックエンドを単一コードで実現している。

---

## Occlusion（RayTracingOcclusionRGS.usf: line 654）

各ピクセル（または Hair サブピクセル）からシャドウレイを発射し可視性を評価する。

```hlsl
// RAY_TRACING_ENTRY_RAYGEN_OR_INLINE(Occlusion)
// → HW RT 時は RGS として実行
// → Inline RT 時は TraceRayInline() を使う CS として実行

{
    uint2 PixelCoord = DispatchThreadIndex.xy + View.ViewRectMin.xy + PixelOffset;

    // 1. GBuffer から WorldPosition・Normal・ShadingModel を取得
    // 2. ライトタイプ別にシャドウレイを生成
    //    - Directional: TraceOcclusionRay() 1本
    //    - Spot/Point:  TraceOcclusionRay() × SamplesPerPixel
    //    - Rect:        RayTracingRectLight.ush で複数サンプル
    // 3. [USE_TRANSLUCENT_SHADOW]
    //    半透明オブジェクトは TransmissionSampling で追加レイを発射
    //    → Shadow Mask の透過率を評価
    // 4. RWOcclusionMaskUAV[PixelCoord] = Visibility (0.0〜1.0)
    // 5. RWRayDistanceUAV[PixelCoord] = ヒット距離（デノイザー向け）
}
```

**パーミュテーション:**

| マクロ | 値 | 説明 |
|--------|-----|------|
| `COMPUTESHADER` | 0/1 | 0=RGS / 1=Inline CS |
| `USE_TRANSLUCENT_SHADOW` | 0/1 | 半透明シャドウサポート |
| `HAIR_STRANDS_SHADOW` | 0/1 | Hair Strands サブピクセルシャドウ |
| `SUBSTRATE_GBUFFER_FORMAT` | 0/1 | Substrate GBuffer 形式 |
| `ShadowMaskType` | Opaque / Hair | シャドウマスクの種別 |

---

## 主要パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `TLAS` | `RaytracingAccelerationStructure` | シーン TLAS |
| `RWOcclusionMaskUAV` | `RWTexture2D<float4>` | 出力: 可視性マスク（R=シャドウ）|
| `RWRayDistanceUAV` | `RWTexture2D<float>` | 出力: ヒット距離 |
| `RWSubPixelOcclusionMaskUAV` | `RWTexture2D<float4>` | Hair サブピクセル用マスク |
| `LightingChannelMask` | `uint` | ライティングチャンネルフィルタ |
| `SamplesPerPixel` | `uint` | ピクセルあたりサンプル数（1〜4）|
| `NormalBias` | `float` | 自己交差防止バイアス |
| `TraceDistance` | `float` | 最大トレース距離 |
| `bAcceptFirstHit` | `uint` | 最初のヒットで即採用（高速）|
| `bTransmissionSamplingDistanceCulling` | `uint` | 半透明カリング有効 |

---

## 半透明シャドウ（USE_TRANSLUCENT_SHADOW）

```hlsl
// AnyHit シェーダーで半透明マテリアルの透過率を蓄積
// → TransmissionMeanFreePath に基づいて透過率を計算
// → ShadowMask.rgb = Throughput（透過後の光量比）
```

---

## シャドウ用マテリアルヒットシェーダー（RayTracingMaterialDefaultHitShaders.usf）

| 関数名 | タイプ | 説明 |
|--------|--------|------|
| `OpaqueShadowCHS` | CHS | 不透明シャドウ用（ペイロードに Hit フラグのみ設定）|
| `HiddenMaterialCHS` | CHS | 非表示マテリアル用 Closest Hit |
| `HiddenMaterialAHS` | AHS | 非表示マテリアル用 Any Hit |

```hlsl
// OpaqueShadowCHS: ヒット時に IsMiss フラグを 0 にするだけ
RAY_TRACING_ENTRY_CLOSEST_HIT(OpaqueShadowCHS,
    FPackedMaterialClosestHitPayload, PackedPayload,
    FRayTracingIntersectionAttributes, Attributes)
{
    PackedPayload.SetIsHit();  // 遮蔽あり
}
```

---

## シャドウパイプライン

```
[CPU] RayTracingShadows::RenderRayTracingShadows()
  for each Light:
    ├─ [HW RT] DispatchRays(OcclusionRGS, {W, H, 1})
    └─ [Inline] Dispatch(OcclusionCS, {W/8, H/8, 1})
            ↓
  OcclusionMask テクスチャ（R=Penumbra, G=シャドウ距離）
            ↓
  IScreenSpaceDenoiser::DenoiseShadows()
  → デノイズ済みシャドウマスク → ライティングパスで使用
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RayTracing.Shadows` | -1 | -1=自動 / 0=無効 / 1=有効 |
| `r.RayTracing.Shadow.SamplesPerPixel` | 1 | ピクセルあたりサンプル数 |
| `r.RayTracing.Shadow.NormalBias` | 0.1 | セルフシャドウバイアス |
| `r.RayTracing.Shadow.AcceptFirstHit` | 0 | 最初のヒットで即採用 |
