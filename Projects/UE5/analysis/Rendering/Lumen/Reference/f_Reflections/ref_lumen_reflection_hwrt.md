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

## RenderLumenHardwareRayTracingReflections

```cpp
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

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（TLAS / Far Field TLAS アクセス）|
| `SceneTextures` | `const FSceneTextureParameters&` | HZB スクリーントレース用テクスチャ |
| `View` | `const FViewInfo&` | カメラビュー |
| `TracingParameters` | `const FLumenCardTracingParameters&` | Surface Cache アトラス・トレース共通パラメータ |
| `ReflectionTracingParameters` | `const FLumenReflectionTracingParameters&` | 反射固有トレースパラメータ（RayBuffer, UAV 等）|
| `ReflectionTileParameters` | `const FLumenReflectionTileParameters&` | タイル分類・Indirect Args |
| `RadianceCacheParameters` | `FRadianceCacheInterpolationParameters` | ミス時のフォールバック照明 |
| `MaxVoxelTraceDistance` | `float` | SW RT の Voxel トレース最大距離（HW RT 開始距離の参考値）|
| `bUseRadianceCache` | `bool` | Radiance Cache フォールバックを使うか |
| `ComputePassFlags` | `ERDGPassFlags` | AsyncCompute / Compute の選択フラグ |

### 内部処理フロー

1. **HZB スクリーントレース（共通パス）**
   ```cpp
   if (LumenReflections::UseScreenTraces(View)) {
       TraceReflectionScreenTexturesCS(GraphBuilder, View, SceneTextures,
           ReflectionTracingParameters, ...);
       // SW RT と同じ FReflectionGenerateRaysCS を先に実行済み
   }
   ```

2. **コンパクト化（Near Field）**
   ```cpp
   FCompactedReflectionTraceParameters CompactedTraceParams =
       LumenReflections::CompactTraces(
           GraphBuilder, View, TracingParameters, ReflectionTracingParameters,
           ReflectionTileParameters, true, CardTraceEndDistance, MaxTraceDistance,
           ComputePassFlags, ETraceCompactionMode::Default,
           LumenReflections::UseHitLighting(View, DiffuseMethod) /*bSortByMaterial*/);
   ```

3. **HW RT Dispatch（Near Field）**
   ```cpp
   // Inline RT or RayGen どちらかを選択
   if (IsInlineSupported(View)) {
       // FLumenReflectionHardwareRayTracingCS（Compute + TraceRayInline）
       DispatchInlineRayTrace(GraphBuilder, CompactedTraceParams, ...);
   } else {
       // FLumenReflectionHardwareRayTracingRGS（RayGen Shader）
       DispatchRayGenShader(GraphBuilder, CompactedTraceParams, ...);
   }
   // ペイロード: LumenMinimal（UseHitLighting=false）または Hit Lighting
   ```

4. **Hit Lighting パス（UseHitLighting=true の場合）**
   ```cpp
   // ヒット点のマテリアル ID を TraceMaterialId に書き込み
   FCompactedReflectionTraceParameters HitLightingCompacted =
       LumenReflections::CompactTraces(..., ETraceCompactionMode::HitLighting, true);
   // 別パスでフルマテリアル評価 + 全光源の Direct Lighting 計算
   EvaluateHitLighting(GraphBuilder, HitLightingCompacted, ...);
   ```

5. **Far Field トレース（オプション）**
   ```cpp
   if (LumenReflections::UseFarField(ViewFamily)) {
       FCompactedReflectionTraceParameters FarFieldCompacted =
           LumenReflections::CompactTraces(..., ETraceCompactionMode::FarField, false);
       // Far Field TLAS で追加トレース
       DispatchFarFieldRayTrace(GraphBuilder, FarFieldCompacted, ...);
   }
   ```

6. **Radiance Cache フォールバック**
   ```cpp
   if (bUseRadianceCache) {
       ApplyRadianceCacheToReflections(GraphBuilder, RadianceCacheParameters, ...);
   }
   ```

### 使用箇所

- [[ref_lumen_reflections]] — `r.Lumen.HardwareRayTracing = 1` の場合に `RenderLumenReflections()` から呼ばれる

---

## シェーダークラス

```cpp
// Inline RT バリアント（Compute Shader + TraceRayInline）
class FLumenReflectionHardwareRayTracingCS
    : public FLumenHardwareRayTracingShaderBase {
    DECLARE_LUMEN_RAYTRACING_SHADER(
        FLumenReflectionHardwareRayTracingCS,
        Lumen.Reflection.HardwareRayTracing)

    // FBasePermutationDomain を継承（[[ref_lumen_hwrt_common]] 参照）
    // + FLightingModePermutation（Surface Cache / Hit Lighting の切り替え）
};

// RayGen バリアント
class FLumenReflectionHardwareRayTracingRGS
    : public FLumenHardwareRayTracingShaderBase {
    DECLARE_LUMEN_RAYTRACING_SHADER(
        FLumenReflectionHardwareRayTracingRGS,
        Lumen.Reflection.HardwareRayTracing)
};
```

### 追加パーミュテーション

| パーミュテーション | 説明 |
|------------------|------|
| `FLightingModePermutation` | `Surface Cache`（0）vs `Hit Lighting`（1）|
| `FAvoidSelfIntersectionsMode` | 自己交差回避モード（[[ref_lumen_hwrt_common]] の `FBasePermutationDomain` 継承）|

### 使用箇所

- [[ref_lumen_reflection_hwrt]] — `IsInlineSupported()` の結果に応じて CS / RGS を選択

---

## Hit Lighting モードの詳細

> [!note]- UseHitLighting / IsHitLightingForceEnabled — Hit Lighting 使用判定
>
> ```cpp
> bool LumenReflections::UseHitLighting(
>     const FViewInfo& View,
>     EDiffuseIndirectMethod DiffuseIndirectMethod);
>
> bool LumenReflections::IsHitLightingForceEnabled(
>     const FViewInfo& View,
>     EDiffuseIndirectMethod DiffuseIndirectMethod);
> ```
>
> **UseHitLighting**: `r.Lumen.HardwareRayTracing.LightingMode = 1` かつ DiffuseIndirectMethod が Lumen の場合
>
> **IsHitLightingForceEnabled**: CVar で強制有効化されているか（演算コスト無視の映像品質モード）
>
> **Hit Lighting 時の処理フロー**:
> 1. ヒット点のマテリアル ID を `TraceMaterialId` に書き込み
> 2. `ETraceCompactionMode::HitLighting` でコンパクト化（マテリアル別ソート）
> 3. 別パスでフルマテリアル評価 + 全光源の Direct Lighting 計算
>
> **使用箇所**: [[ref_lumen_reflection_hwrt]] — HW RT 前にペイロード種別の選択に使用

---

## マルチバウンス反射

```cpp
// r.Lumen.Reflections.MaxBounces > 1 の場合:
// → 反射ヒット点から再度 HW RT レイを発射
// → FLumenReflectionTracingParameters::MaxReflectionBounces で制御
// → 各バウンスで ReflectionPass カウンタをインクリメント

// 屈折バウンス（透明マテリアル）
// FLumenReflectionTracingParameters::MaxRefractionBounces で制御
// → 透明ヒット点で屈折ベクトルを計算して追加レイを発射
```

### 使用箇所

- [[ref_ray_traced_translucency]] — 透明マテリアルの多段レイヤー処理と連携

---

## SW RT との差異

| 比較項目 | SW RT（TraceReflections）| HW RT（RenderLumenHardwareRayTracingReflections）|
|---------|---------|---------|
| 精度 | SDF 近似誤差あり | ジオメトリに対して正確 |
| フォールバック | HZB → Mesh SDF → Global SDF | HZB スクリーン → HW RT のみ |
| GPU コスト | 低め | 高め（BLAS トラバーサル）|
| Hit Lighting | 不可 | 可（UseHitLighting() = true 時）|
| MultiLayer 屈折 | 不可 | 可（透明マテリアル重ね合わせ）|
| Far Field | Global SDF 到達範囲 | Far Field TLAS で遠景もカバー |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.Reflections.HardwareRayTracing` | 1 | HW RT 反射の有効/無効 |
| `r.Lumen.Reflections.HardwareRayTracing.LightingMode` | 0 | 0=Surface Cache, 1=Hit Lighting |
| `r.Lumen.Reflections.HardwareRayTracing.MinTraceDistance` | 0.0 | HW RT 最小トレース距離 |
| `r.Lumen.Reflections.HardwareRayTracing.MaxTraceDistance` | 1000000.0 | HW RT 最大トレース距離 |
| `r.Lumen.Reflections.HardwareRayTracing.SkipBackfaceHits` | 1 | 裏面ヒットを無視するか |
| `r.Lumen.Reflections.HardwareRayTracing.Retrace.FarField` | 1 | Far Field 再トレースの有効/無効 |
