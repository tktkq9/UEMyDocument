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

## 主要エントリポイント

```cpp
// 反射レイのトレース全体パス
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

---

## トレースパイプライン

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
  │   └─ IndirectArgs (ClearIndirect / ResolveIndirect / TracingIndirect) を生成
  │
  ├─ [HZB スクリーンスペーストレース]
  │   UseScreenTraces() = true の場合:
  │   TraceReflectionScreenTextures()
  │   → HZBを階層的に下降しながら反射先を探索
  │   → ヒット: TraceHit テクスチャに距離を書き込み
  │   → ミス: 次段(Mesh SDF)へ
  │
  ├─ [Mesh SDF トレース]
  │   r.Lumen.Reflections.TraceMeshSDFs = 1 の場合:
  │   TraceReflectionMeshSDFs()
  │   → コンパクト化（CompactTraces）してから Indirect Dispatch
  │   → 個別 Mesh の Signed Distance Field でトレース
  │
  ├─ [Global SDF トレース]
  │   TraceReflectionGlobalSDF()
  │   → 粗い Global SDF（ボクセルベース）で長距離をカバー
  │
  ├─ [Far Field トレース]
  │   UseFarField() = true かつ未解決ピクセルが残る場合:
  │   TraceReflectionFarField()
  │
  ├─ [Radiance Cache フォールバック]
  │   UseRadianceCache() = true の場合:
  │   → 粗面（高 Roughness）のレイを Radiance Cache で補間
  │   → MinRoughness〜MaxRoughness でブレンド
  │
  └─ [結果書き込み]
      TraceRadiance / TraceHit テクスチャを更新
      → ResolveReflections() で解像度復元・テンポラル合成
```

---

## GenerateReflectionRays

```cpp
// レイバッファ生成シェーダー（Compute Shader）
class FReflectionGenerateRaysCS : public FGlobalShader {
    // 各ピクセルの GBuffer 法線・Roughness から
    // GGX 重み付き反射レイ方向 (RayBuffer) を生成
    // UseJitter = true の場合 Blue Noise でジッターを加える
    // DownsampledDepth / DownsampledClosureIndex も同時に生成
};
```

---

## CompactTraces との連携

```cpp
// Mesh SDF トレース前のコンパクト化
FCompactedReflectionTraceParameters CompactedTraceParams =
    LumenReflections::CompactTraces(
        GraphBuilder, View,
        TracingParameters,
        ReflectionTracingParameters,
        ReflectionTileParameters,
        bCullByDistanceFromCamera,
        CompactionTracingEndDistance,
        CompactionMaxTraceDistance,
        ComputePassFlags,
        ETraceCompactionMode::Default,  // 通常トレース
        bSortByMaterial);               // マテリアル別ソート（HitLighting時）
```

---

## テンポラル蓄積

```cpp
// テンポラル蓄積パス（r.Lumen.Reflections.Temporal = 1 の場合）
class FReflectionTemporalCS : public FGlobalShader {
    // 前フレームの ReflectionHistory と現フレームの TraceRadiance をブレンド
    // MaxFramesAccumulated = 12.0f
    // 動き検出によりゴースティングを抑制
};
```

---

## ResolveReflections

```cpp
// ダウンサンプル済みトレース結果を元の解像度に復元
class FReflectionResolveCS : public FGlobalShader {
    // ReflectionTracingViewMin / ViewSize を使ってビューポートにマッピング
    // DownsampleFactor > 1 の場合は隣接ピクセルの補間も行う
};
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
r.Lumen.HardwareRayTracing = 1 → RenderLumenHardwareRayTracingReflections()（LumenReflectionHardwareRayTracing.cpp）
```
