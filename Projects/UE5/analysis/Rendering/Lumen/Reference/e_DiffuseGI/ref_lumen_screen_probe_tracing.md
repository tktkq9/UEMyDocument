# リファレンス：LumenScreenProbeTracing.cpp

- グループ: e - Diffuse GI
- 上位: [[e_lumen_diffuse_gi]]
- 関連: [[ref_lumen_screen_probe_gather]] | [[ref_lumen_tracing_utils]] | [[ref_lumen_mesh_sdf_culling]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenScreenProbeTracing.cpp`

---

## 概要

Screen Probe の**ソフトウェアレイトレーシング（SW RT）**実装ファイル。  
各プローブから HZB スクリーントレース → Mesh SDF → Global SDF の順で  
フォールバックしながら Radiance をサンプリングする。

---

## TraceScreenProbes（エントリポイント）

```cpp
void TraceScreenProbes(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    bool bTraceMeshObjects,
    const FSceneTextures& SceneTextures,
    FRDGTextureRef LightingChannelsTexture,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters& RadianceCacheParameters,
    FScreenProbeParameters& ScreenProbeParameters,
    FLumenMeshSDFGridParameters& MeshSDFGridParameters,
    ERDGPassFlags ComputePassFlags);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（Global SDF アクセスに使用）|
| `View` | `const FViewInfo&` | カメラビュー |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | Surface Cache アトラス（ヒット時のサンプル先）|
| `bTraceMeshObjects` | `bool` | Mesh SDF トレースを行うか（`r.Lumen.ScreenProbeGather.TraceMeshSDFs` に対応）|
| `SceneTextures` | `const FSceneTextures&` | GBuffer / HZB テクスチャ（スクリーントレース用）|
| `LightingChannelsTexture` | `FRDGTextureRef` | ライティングチャンネルマスク（プリミティブ単位）|
| `RadianceCacheParameters` | `FRadianceCacheInterpolationParameters` | ミス時に Radiance Cache から補間 |
| `ScreenProbeParameters` | `FScreenProbeParameters&` | プローブパラメータ（TraceRadiance / TraceHit UAV に書き込み）|
| `MeshSDFGridParameters` | `FLumenMeshSDFGridParameters&` | フラスタムグリッドカリング結果 |
| `ComputePassFlags` | `ERDGPassFlags` | AsyncCompute / Compute の選択フラグ |

### 内部処理フロー

1. **重要度サンプリング（IS）パス**
   ```cpp
   if (UseImportanceSampling(View)) {
       // [[ref_lumen_screen_probe_importance]] GenerateImportanceSamplingRays() を呼ぶ
       // → StructuredImportanceSampledRayInfosForTracing にレイ方向を格納
   }
   ```

2. **HZB スクリーントレース**
   ```cpp
   // r.Lumen.ScreenProbeGather.ScreenTraces が有効な場合
   // → 前フレームの Depth / Color バッファを HZB でトレース
   // → ヒット: PrevSceneColorTexture から Radiance を取得
   // → TraceRadiance / TraceHit UAV に部分的に書き込み
   ScreenTraceScreenProbes(GraphBuilder, View, SceneTextures, ScreenProbeParameters, ...);
   ```

3. **Mesh SDF トレース（bTraceMeshObjects=true の場合）**
   ```cpp
   // CompactTraces() でスクリーントレースのミスを集約
   FCompactedTraceParameters CompactedTraceParameters = LumenScreenProbeGather::CompactTraces(
       GraphBuilder, View, ScreenProbeParameters,
       /*bCullByDistance*/ true, CardTraceEndDistanceFromCamera, MaxTraceDistance, ...);
   // MeshSDFGridParameters から近傍 Mesh SDF オブジェクトをトレース
   // → ヒット: Surface Cache の FinalLightingAtlas をサンプリング
   TraceScreenProbesMeshSDFs(GraphBuilder, Scene, View, FrameTemporaries,
       ScreenProbeParameters, MeshSDFGridParameters, CompactedTraceParameters, ...);
   ```

4. **Global SDF トレース**
   ```cpp
   // さらにミスしたレイを Global SDF（低解像度）でトレース
   // → ヒット: Surface Cache をサンプリング
   // → ミス: Radiance Cache または Skylight を使用
   TraceScreenProbesGlobalSDF(GraphBuilder, Scene, View, FrameTemporaries,
       ScreenProbeParameters, RadianceCacheParameters, CompactedTraceParameters, ...);
   ```

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — `RenderLumenScreenProbeGather()` から SW RT パスとして呼ばれる
- [[ref_lumen_screen_probe_hwrt]] — HW RT が無効または非対応時のフォールバックとして呼ばれる

---

## FCompactedTraceParameters

コンパクト化後のトレースデータバッファ群。  
`CompactTraces()` が返す構造体で、不要なトレースを除いたリストを格納する。

```cpp
struct FCompactedTraceParameters {
    FRDGBufferRef CompactedTraceTexelAllocator; // 有効トレース数カウンタ
    FRDGBufferRef CompactedTraceTexelData;      // 有効トレースのプローブID・方向インデックス
    FRDGBufferRef IndirectArgs;                 // Indirect Dispatch Args
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `CompactedTraceTexelAllocator` | `FRDGBufferRef` | Atomic カウンタ（有効トレースの総数）|
| `CompactedTraceTexelData` | `FRDGBufferRef` | 各エントリに { ProbeIndex, RayIndex } を格納 |
| `IndirectArgs` | `FRDGBufferRef` | `CompactedTraceTexelAllocator` から生成した Indirect Dispatch Args |

### 使用箇所

- [[ref_lumen_screen_probe_tracing]] — Mesh SDF / Global SDF トレースの Dispatch に使用
- [[ref_lumen_screen_probe_hwrt]] — HW RT Dispatch の前にコンパクト化を実行

---

## SW RT トレースの 3 層フォールバック構造

```
各 Screen Probe について:

1. [HZB Screen Trace]
   → 前フレームの Depth / Color バッファを HZB でトレース
   → ヒット: 前フレームのシーンカラーから Radiance を取得
   → ミス: 次のレイヤーへ

2. [Mesh SDF Trace]（bTraceMeshObjects = true の場合）
   → CullMeshObjectsToViewGrid() で構築したグリッドから
     近傍 Mesh SDF オブジェクトをトレース
   → ヒット: Surface Cache の FinalLightingAtlas をサンプリング
   → ミス / 射程外: 次のレイヤーへ

3. [Global SDF Trace / Radiance Cache]
   → Global SDF（低解像度）でトレース
   → ヒット: Surface Cache をサンプリング
   → ミス: Radiance Cache（クリップマップ補間）または Skylight を使用
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.ScreenProbeGather.TraceMeshSDFs` | 1 | Mesh SDF トレースの有効/無効 |
| `r.Lumen.ScreenProbeGather.ScreenTraces` | 1 | HZB スクリーントレースの有効/無効 |
| `r.Lumen.ScreenProbeGather.ScreenTraces.HZBOcclusionTest` | 1 | HZB でオクルージョンテスト |
| `r.Lumen.ScreenProbeGather.ScreenTraces.MaxRayIntensity` | 40.0 | スクリーントレースの最大輝度クランプ |
| `r.Lumen.ScreenProbeGather.OverrideSkyColor` | — | スカイカラーを上書き（デバッグ用）|

---

## HZB Screen Trace の仕組み

```
FLumenHZBScreenTraceParameters（[[ref_lumen_tracing_utils]] 参照）:
  PrevSceneColorTexture  : 前フレームのシーンカラー（時間再投影）
  HistorySceneDepth      : 前フレームの深度バッファ
  HZBParameters          : Hierarchical Z Buffer（ミップチェーン）

トレース手順:
  1. プローブの位置・方向からスクリーン空間レイを生成
  2. HZB の粗いミップで高速にミス判定 → 細かいミップで精密化
  3. ヒット時: PrevSceneColorTexture から UV でサンプリング
  4. ScreenTraceNoFallbackThicknessScale でハード面の厚みを調整
     → 値が大きいほど厚いサーフェスとして扱い、裏面ヒットを避ける
```
