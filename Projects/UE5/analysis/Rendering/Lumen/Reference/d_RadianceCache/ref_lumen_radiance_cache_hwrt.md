# リファレンス：LumenRadianceCacheHardwareRayTracing.cpp

- グループ: d - Radiance Cache
- 上位: [[d_lumen_radiance_cache]]
- 関連: [[ref_lumen_radiance_cache]] | [[ref_lumen_radiance_cache_internal]] | [[ref_lumen_hwrt_common]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenRadianceCacheHardwareRayTracing.cpp`

---

## 概要

Radiance Cache プローブの HW RT トレース実装ファイル。  
`RenderLumenHardwareRayTracingRadianceCache()` がエントリポイントで、  
各プローブのトレース方向に対して RT レイを発射し、Surface Cache から Radiance をサンプリングする。  
`#if RHI_RAYTRACING` ブロック内にのみ実装される。

---

## エントリポイント関数

### RenderLumenHardwareRayTracingRadianceCache（新 API）

複数の Radiance Cache を同時に処理する新しいエントリポイント。  
`UpdateRadianceCaches()` から呼ばれ、各キャッシュのトレースを GPU 上でオーバーラップさせる。

```cpp
namespace LumenRadianceCache {
    void RenderLumenHardwareRayTracingRadianceCache(
        FRDGBuilder& GraphBuilder,
        const FScene* Scene,
        const FLumenSceneFrameTemporaries& FrameTemporaries,
        const TInlineArray<FUpdateInputs>& InputArray,
        TInlineArray<FUpdateOutputs>& OutputArray,
        const TInlineArray<FRadianceCacheSetup>& SetupOutputArray,

        // プローブごとのトレースタイルデータ（どの方向をトレースするか）
        const TInlineArray<FRDGBufferRef>& ProbeTraceTileAllocatorArray,
        const TInlineArray<FRDGBufferRef>& ProbeTraceTileDataArray,

        // プローブのワールド座標・インデックス
        const TInlineArray<FRDGBufferRef>& ProbeTraceDataArray,

        // HW RT レイアロケータ（レイバッファ管理）
        const TInlineArray<FRDGBufferRef>& HardwareRayTracingRayAllocatorBufferArray,

        // Indirect Dispatch Args
        const TInlineArray<FRDGBufferRef>& TraceProbesIndirectArgsArray,

        ERDGPassFlags ComputePassFlags);
}
```

### RenderLumenHardwareRayTracingRadianceCache_REMOVE（旧 API）

旧来の単一キャッシュ API。将来的に削除予定（`_REMOVE` サフィックスが目印）。  
新しいコードでは上記の新 API を使用する。

```cpp
namespace LumenRadianceCache {
    void RenderLumenHardwareRayTracingRadianceCache_REMOVE(
        FRDGBuilder& GraphBuilder,
        const FScene* Scene,
        const FSceneTextureParameters& SceneTextures,
        const FViewInfo& View,
        const FLumenCardTracingParameters& TracingParameters,
        const LumenRadianceCache::FRadianceCacheInterpolationParameters& RadianceCacheParameters,
        FRadianceCacheConfiguration Configuration,
        int32 MaxNumProbes,
        int32 MaxProbeTraceTileResolution,
        FRDGBufferRef ProbeTraceData,
        FRDGBufferRef ProbeTraceTileData,
        FRDGBufferRef ProbeTraceTileAllocator,
        FRDGBufferRef TraceProbesIndirectArgs,
        FRDGBufferRef HardwareRayTracingRayAllocatorBuffer,
        FRDGBufferRef RadianceCacheHardwareRayTracingIndirectArgs,
        FRDGTextureUAVRef RadianceProbeAtlasTextureUAV,
        FRDGTextureUAVRef DepthProbeTextureUAV,
        ERDGPassFlags ComputePassFlags);
}
```

---

## トレーストークン（ProbeTraceTile）

```
ProbeTraceTileAllocator: トレースするタイル数カウンタ
ProbeTraceTileData:     各タイルのデータ
  → タイルごとに 8×8 = 64 方向のレイを束ねて Dispatch
  → TRACE_TILE_SIZE_2D = 8（LumenRadianceCacheInternal.h で定義）

ProbeTraceData:
  → 各プローブのワールド座標・プローブ ID・クリップマップ番号

HardwareRayTracingRayAllocatorBuffer:
  → レイバッファの動的割り当てカウンタ
  → Inline RT の場合、1 スレッドで複数レイを処理
```

---

## Radiance Cache HW RT トレースの流れ

```
RenderLumenHardwareRayTracingRadianceCache()
  │
  ├─ [ProbeTraceTile の生成]
  │   プローブのトレース方向（半球 NxN 分割）をタイルに分解
  │   → ProbeTraceTileAllocator / ProbeTraceTileData に書き込み
  │
  ├─ [RT Dispatch]
  │   各タイル × 64 方向に HW RT レイを発射
  │   ├─ ペイロード: LumenMinimal（高速シャドウ判定）
  │   ├─ ヒット時: Surface Cache の FinalLightingAtlas をサンプリング
  │   └─ ミス時: スカイライトを使用
  │
  ├─ [Far Field]
  │   Configuration.bFarField = true の場合:
  │   → 近距離 TLAS のミス後に Far Field TLAS でもトレース
  │
  └─ [RadianceProbeAtlasTexture に書き込み]
        → LumenRadianceCache.cpp でフィルタ・SH 射影して最終アトラスへ
```

---

## SW RT との比較

| 比較項目 | SW RT（Surface Cache サンプリング）| HW RT |
|---------|------|------|
| 精度 | Surface Cache の解像度に依存 | ジオメトリに対して正確 |
| ヒット照明 | FinalLightingAtlas のサンプリング | UseHitLighting=true でフルマテリアル評価も可能 |
| GPU コスト | 低い（テクスチャフェッチのみ）| 高い（BLAS トラバーサル）|
| 有効条件 | 常に利用可 | RT 対応 GPU のみ |
