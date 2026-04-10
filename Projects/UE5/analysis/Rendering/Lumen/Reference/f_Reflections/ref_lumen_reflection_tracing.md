# リファレンス：LumenReflectionTracing.cpp

- グループ: f - Reflections
- 上位: [[f_lumen_reflections]]
- 関連: [[ref_lumen_reflections]] | [[ref_lumen_reflection_hwrt]] | [[ref_lumen_tracing_utils]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenReflectionTracing.cpp`

---

## 概要

Lumen Reflections の **SW（Software）RT トレース実装**ファイル。  
レイバッファ生成・タイル分類・コンパクト化・HZB/Mesh SDF/Global SDF の多段フォールバックトレース・  
テンポラル蓄積までの一連のパスを担う。

---

## TraceReflections（エントリポイント）

```cpp
void TraceReflections(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FLumenCardTracingParameters& TracingParameters,
    const FLumenReflectionTracingParameters& ReflectionTracingParameters,
    const FLumenReflectionTileParameters& ReflectionTileParameters,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters& RadianceCacheParameters,
    bool bUseRadianceCache,
    ERDGPassFlags ComputePassFlags);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（Global SDF / Far Field アクセス）|
| `View` | `const FViewInfo&` | カメラビュー |
| `TracingParameters` | `const FLumenCardTracingParameters&` | Surface Cache アトラス・トレース共通パラメータ |
| `ReflectionTracingParameters` | `const FLumenReflectionTracingParameters&` | 反射固有トレースパラメータ（RayBuffer, TraceRadiance UAV 等）|
| `ReflectionTileParameters` | `const FLumenReflectionTileParameters&` | タイル分類・Indirect Args |
| `RadianceCacheParameters` | `FRadianceCacheInterpolationParameters` | ミス時のフォールバック照明 |
| `bUseRadianceCache` | `bool` | Radiance Cache フォールバックを使うか |
| `ComputePassFlags` | `ERDGPassFlags` | AsyncCompute / Compute の選択フラグ |

### 内部処理フロー

1. **レイバッファ生成**
   ```cpp
   // GBuffer の法線・Roughness から GGX 重み付き反射レイ方向を生成
   // UseJitter = true の場合 BlueNoise でジッターを加える
   GenerateReflectionRaysCS(GraphBuilder, View, ReflectionTracingParameters, ...);
   // → RayBuffer: Texture2D<float4>（xyz=方向, w=GGX PDF）
   // → DownsampledDepth / DownsampledClosureIndex も同時生成
   ```

2. **タイル分類**
   ```cpp
   // タイルごとにトレースが必要かを分類
   ClassifyReflectionTilesCS(GraphBuilder, ReflectionTracingParameters, ReflectionTileParameters, ...);
   // → LumenTileBitmask に分類を書き込み
   // → ClearIndirectArgs / ResolveIndirectArgs / TracingIndirectArgs を生成
   ```

3. **HZB スクリーンスペーストレース**
   ```cpp
   if (LumenReflections::UseScreenTraces(View)) {
       TraceReflectionScreenTexturesCS(GraphBuilder, View, ReflectionTracingParameters, ...);
       // → HZBを階層的に下降しながら反射先を探索
       // → ヒット: TraceHit テクスチャに距離を書き込み
   }
   ```

4. **Mesh SDF トレース**
   ```cpp
   if (r.Lumen.Reflections.TraceMeshSDFs && bTraceMeshObjects) {
       FCompactedReflectionTraceParameters Compacted = LumenReflections::CompactTraces(
           GraphBuilder, View, TracingParameters, ReflectionTracingParameters,
           ReflectionTileParameters, true, CardTraceEndDistance, MaxTraceDistance,
           ComputePassFlags, ETraceCompactionMode::Default, false);
       TraceReflectionMeshSDFsCS(GraphBuilder, Compacted, ...);
   }
   ```

5. **Global SDF トレース**
   ```cpp
   // さらにミスしたレイを粗い Global SDF でカバー
   TraceReflectionGlobalSDFCS(GraphBuilder, Scene, View, TracingParameters,
       ReflectionTracingParameters, ReflectionTileParameters, ...);
   ```

6. **Far Field トレース（オプション）**
   ```cpp
   if (LumenReflections::UseFarField(ViewFamily)) {
       FCompactedReflectionTraceParameters FarFieldCompacted = LumenReflections::CompactTraces(
           ..., ETraceCompactionMode::FarField, false);
       TraceReflectionFarFieldCS(GraphBuilder, FarFieldCompacted, ...);
   }
   ```

7. **Radiance Cache フォールバック**
   ```cpp
   if (bUseRadianceCache) {
       // 粗面（高 Roughness）のミスレイを Radiance Cache で補間
       // MinRoughness〜MaxRoughness でブレンド
       ApplyRadianceCacheToReflections(GraphBuilder, RadianceCacheParameters, ...);
   }
   ```

8. **Resolve + テンポラル蓄積**
   ```cpp
   // FReflectionResolveCS: ダウンサンプル結果を元解像度に復元
   ResolveReflectionsCS(GraphBuilder, ReflectionTracingParameters, ...);
   // FReflectionTemporalCS: 前フレームとブレンド（MaxFramesAccumulated=12）
   if (r.Lumen.Reflections.Temporal) {
       ReflectionTemporalCS(GraphBuilder, ...);
   }
   ```

### 使用箇所

- [[ref_lumen_reflections]] — `r.Lumen.HardwareRayTracing = 0` の場合に `RenderLumenReflections()` から呼ばれる

---

## GenerateReflectionRays

```cpp
// レイバッファ生成 Compute Shader
class FReflectionGenerateRaysCS : public FGlobalShader {
    // 各ピクセルの GBuffer.Normal / Roughness から
    // GGX 重み付き反射レイ方向 (RayBuffer) を生成
    // UseJitter = true の場合 Blue Noise でジッターを加える
    // DownsampledDepth / DownsampledClosureIndex も同時に生成
};
```

### 使用箇所

- [[ref_lumen_reflection_tracing]] — `TraceReflections()` ステップ1で呼ばれる
- [[ref_lumen_reflection_hwrt]] — HW RT パスでも同じ `FReflectionGenerateRaysCS` を使用

---

## FReflectionTemporalCS / FReflectionResolveCS

> [!note]- FReflectionTemporalCS — テンポラル蓄積シェーダー
>
> ```cpp
> class FReflectionTemporalCS : public FGlobalShader { ... };
> ```
>
> 前フレームの `ReflectionHistory` と現フレームの `TraceRadiance` をブレンド。  
> `MaxFramesAccumulated = 12.0f`（`r.Lumen.Reflections.Temporal.MaxFramesAccumulated`）。  
> 動き検出によりゴースティングを抑制。
>
> **使用箇所**: [[ref_lumen_reflection_tracing]] — `TraceReflections()` の最終ステップ（テンポラル有効時）

> [!note]- FReflectionResolveCS — 解像度復元シェーダー
>
> ```cpp
> class FReflectionResolveCS : public FGlobalShader { ... };
> ```
>
> ダウンサンプル済みトレース結果を元の解像度に復元する。  
> `ReflectionTracingViewMin / ViewSize` を使ってビューポートにマッピング。  
> `DownsampleFactor > 1` の場合は隣接ピクセルの補間も行う。
>
> **使用箇所**: [[ref_lumen_reflection_tracing]] — テンポラル蓄積の前にResolveを実行

---

## トレースパイプライン全体

```
TraceReflections()
  │
  ├─ [レイバッファ生成]
  │   GenerateReflectionRays() ← GBuffer の法線・ラフネスから GGX 重み付きレイ方向を生成
  │   RayBuffer: Texture2D<float4>（xyz=方向, w=GGX PDF）
  │
  ├─ [タイル分類]
  │   ClassifyReflectionTiles()
  │   ├─ LumenTileBitmask (Texture2DArray) にカテゴリを書き込み
  │   └─ IndirectArgs (Clear / Resolve / Tracing) を生成
  │
  ├─ [HZB スクリーンスペーストレース]
  │   UseScreenTraces() = true の場合
  │   → HZB 階層を下降しながら反射先を探索
  │
  ├─ [Mesh SDF トレース]
  │   r.Lumen.Reflections.TraceMeshSDFs = 1 の場合
  │   → CompactTraces(Default) → Indirect Dispatch
  │
  ├─ [Global SDF トレース]
  │   → 粗い Global SDF（ボクセルベース）で長距離をカバー
  │
  ├─ [Far Field トレース]
  │   UseFarField() = true かつ未解決ピクセルが残る場合
  │   → CompactTraces(FarField) → Far Field TLAS でトレース
  │
  ├─ [Radiance Cache フォールバック]
  │   UseRadianceCache() = true の場合
  │   → 粗面（高 Roughness）のレイを Radiance Cache で補間
  │
  └─ [Resolve + テンポラル蓄積]
      FReflectionResolveCS → FReflectionTemporalCS
```

---

## 主要 CVar（LumenReflectionTracing.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.Reflections.HZB.MaxIterations` | 64 | HZB スクリーントレースの最大反復数 |
| `r.Lumen.Reflections.ScreenTraces.HZBTraversalOctahedralSolidAngle` | 0.01 | HZB 終了判定の立体角閾値 |
| `r.Lumen.Reflections.SortTiles` | 1 | タイルのソートを行うか |
| `r.Lumen.Reflections.SortByMaterial` | 1 | HW RT 時のマテリアル別ソート |
| `r.Lumen.Reflections.FarField` | 1 | Far Field トレースの有効/無効 |
| `r.Lumen.Reflections.FarField.MaxTraceDistance` | 1000000.0 | Far Field 最大トレース距離 |
| `r.Lumen.Reflections.FarField.ReferencePos` | 0 | Far Field 基準位置のオフセット |

---

## SW RT / HW RT の切り替え

```
r.Lumen.HardwareRayTracing = 0 → TraceReflections()（このファイル）が使われる
r.Lumen.HardwareRayTracing = 1 → RenderLumenHardwareRayTracingReflections()
                                  （LumenReflectionHardwareRayTracing.cpp）
```
