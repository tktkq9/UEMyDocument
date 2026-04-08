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
    bool bTraceMeshObjects,                // Mesh SDF トレースを行うか
    const FSceneTextures& SceneTextures,
    FRDGTextureRef LightingChannelsTexture,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters& RadianceCacheParameters,
    FScreenProbeParameters& ScreenProbeParameters,
    FLumenMeshSDFGridParameters& MeshSDFGridParameters,
    ERDGPassFlags ComputePassFlags);
```

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

## FCompactedTraceParameters の役割

```cpp
// トレースが必要なプローブ・方向を距離でカリングしてコンパクト化
FCompactedTraceParameters LumenScreenProbeGather::CompactTraces(
    FRDGBuilder&, const FViewInfo&, const FScreenProbeParameters&,
    bool bCullByDistanceFromCamera,
    float CompactionTracingEndDistanceFromCamera, // カメラからの最大距離
    float CompactionMaxTraceDistance,             // トレースの最大距離
    bool bCompactForSkyApply,                     // スカイ適用用のコンパクト化か
    ERDGPassFlags);
// → CompactedTraceTexelAllocator / CompactedTraceTexelData に書き込み
// → 遠距離/不要なトレースをスキップして Dispatch 数を削減
```

---

## HZB Screen Trace の仕組み

```
FLumenHZBScreenTraceParameters:
  PrevSceneColorTexture  : 前フレームのシーンカラー（時間再投影）
  HistorySceneDepth      : 前フレームの深度バッファ
  HZBParameters          : Hierarchical Z Buffer（ミップチェーン）

トレース手順:
  1. プローブの位置・方向からスクリーン空間レイを生成
  2. HZB の粗いミップで高速にミス判定 → 細かいミップで精密化
  3. ヒット時: PrevSceneColorTexture から UV でサンプリング
  4. ScreenTraceNoFallbackThicknessScale でハード面の厚みを調整
```
