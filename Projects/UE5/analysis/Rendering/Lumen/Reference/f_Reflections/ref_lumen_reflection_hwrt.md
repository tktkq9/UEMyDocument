# リファレンス：LumenReflectionHardwareRayTracing.cpp

- グループ: f - Reflections
- 上位: [[f_lumen_reflections]]
- 関連: [[ref_lumen_reflections]] | [[ref_lumen_reflection_tracing]] | [[ref_lumen_hwrt_common]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenReflectionHardwareRayTracing.cpp`

---

## 概要

Lumen Reflections の **Hardware Ray Tracing（HW RT）実装**ファイル。  
`#if RHI_RAYTRACING` ブロック内のみ有効。  
反射レイを HW RT で発射し、Surface Cache サンプリングまたは Hit Lighting でラジアンスを取得する。

---

## 主要エントリポイント

```cpp
// HW RT 反射トレースのメインエントリ
void RenderLumenHardwareRayTracingReflections(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FSceneTextureParameters& SceneTextures,
    const FViewInfo& View,
    const FLumenCardTracingParameters& TracingParameters,
    const FLumenReflectionTracingParameters& ReflectionTracingParameters,
    const FLumenReflectionTileParameters& ReflectionTileParameters,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters& RadianceCacheParameters,
    float MaxVoxelTraceDistance,
    bool bUseRadianceCache,
    ERDGPassFlags ComputePassFlags);
```

---

## HW RT 反射トレースの流れ

```
RenderLumenHardwareRayTracingReflections()
  │
  ├─ [スクリーンスペーストレース（共通）]
  │   UseScreenTraces() = true の場合:
  │   TraceReflectionScreenTextures() を先に実行（SW RT と共通パス）
  │
  ├─ [コンパクト化]
  │   CompactTraces(..., ETraceCompactionMode::Default)
  │   → HW RT Dispatch 用に有効ピクセルのみリストアップ
  │
  ├─ [HW RT Dispatch（Near Field）]
  │   Inline RT or RayGen どちらかを選択
  │   ├─ IsInlineSupported() = true → Inline RT（Compute Shader 内で TraceRay）
  │   └─ false → RayGen Shader（TraceRayInline 不使用）
  │
  │   ペイロードモード:
  │   ├─ UseHitLighting() = false → LumenMinimal ペイロード（16bytes）
  │   │   ヒット点の FinalLightingAtlas を参照してラジアンス取得
  │   └─ UseHitLighting() = true → Hit Lighting モード
  │       ヒット点でフルマテリアル評価 + 直接光計算
  │       → ETraceCompactionMode::HitLighting でコンパクト化
  │
  ├─ [Far Field（オプション）]
  │   UseFarField() = true かつ Near Field でミスしたレイ:
  │   CompactTraces(..., ETraceCompactionMode::FarField)
  │   → Far Field TLAS で追加トレース
  │
  ├─ [Radiance Cache フォールバック]
  │   bUseRadianceCache = true の場合:
  │   → HW RT でミスしたレイを Radiance Cache で補間
  │
  └─ TraceRadiance / TraceHit テクスチャに書き込み
```

---

## シェーダークラス

```cpp
// HW RT 反射シェーダー（DECLARE_LUMEN_RAYTRACING_SHADER マクロで定義）
class FLumenReflectionHardwareRayTracingCS
    : public FLumenHardwareRayTracingShaderBase {
    // Inline RT バリアント（Compute Shader + TraceRayInline）
    DECLARE_LUMEN_RAYTRACING_SHADER(FLumenReflectionHardwareRayTracingCS, Lumen.Reflection.HardwareRayTracing)
};

class FLumenReflectionHardwareRayTracingRGS
    : public FLumenHardwareRayTracingShaderBase {
    // RayGen バリアント
    DECLARE_LUMEN_RAYTRACING_SHADER(FLumenReflectionHardwareRayTracingRGS, Lumen.Reflection.HardwareRayTracing)
};
```

---

## SW RT との差異

| 比較項目 | SW RT（TraceReflections）| HW RT（RenderLumenHardwareRayTracingReflections）|
|---------|---------|---------|
| 精度 | SDF 近似誤差あり | ジオメトリに対して正確 |
| フォールバック | HZB → Mesh SDF → Global SDF | HZB スクリーン → HW RT のみ |
| GPU コスト | 低め | 高め（BLAS トラバーサル）|
| Hit Lighting | 不可 | 可（UseHitLighting() = true 時）|
| MultiLayer | 不可 | 可（透明マテリアル重ね合わせ）|

---

## Hit Lighting モードの詳細

```cpp
// r.Lumen.HardwareRayTracing.LightingMode = 1 の場合:
bool LumenReflections::UseHitLighting(
    const FViewInfo& View,
    EDiffuseIndirectMethod DiffuseIndirectMethod);

bool LumenReflections::IsHitLightingForceEnabled(
    const FViewInfo& View,
    EDiffuseIndirectMethod DiffuseIndirectMethod);

// Hit Lighting 時の処理:
// 1. ヒット点のマテリアル ID を TraceMaterialId に書き込み
// 2. ETraceCompactionMode::HitLighting でコンパクト化
// 3. 別パスでフルマテリアル評価 + 全光源の Direct Lighting 計算
// → コスト非常に高い（映像制作・ハイエンド向け）
```

---

## マルチバウンス反射

```cpp
// r.Lumen.Reflections.MaxBounces > 1 の場合:
// → 反射ヒット点から再度 HW RT レイを発射
// → FLumenReflectionTracingParameters::MaxReflectionBounces で制御
// → 各バウンスで ReflectionPass カウンタをインクリメント

// 屈折バウンス（透明マテリアル）
// FLumenReflectionTracingParameters::MaxRefractionBounces で制御
```

---

## 主要 CVar（LumenReflectionHardwareRayTracing.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.Reflections.HardwareRayTracing` | 1 | HW RT 反射の有効/無効 |
| `r.Lumen.Reflections.HardwareRayTracing.LightingMode` | 0 | 0=Surface Cache, 1=Hit Lighting |
| `r.Lumen.Reflections.HardwareRayTracing.MinTraceDistance` | 0.0 | HW RT 最小トレース距離 |
| `r.Lumen.Reflections.HardwareRayTracing.MaxTraceDistance` | 1000000.0 | HW RT 最大トレース距離 |
| `r.Lumen.Reflections.HardwareRayTracing.SkipBackfaceHits` | 1 | 裏面ヒットを無視するか |
| `r.Lumen.Reflections.HardwareRayTracing.Retrace.FarField` | 1 | Far Field 再トレースの有効/無効 |
