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

## RenderLumenHardwareRayTracingRadianceCache（新 API）

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
        const TInlineArray<FRDGBufferRef>& ProbeTraceTileAllocatorArray,
        const TInlineArray<FRDGBufferRef>& ProbeTraceTileDataArray,
        const TInlineArray<FRDGBufferRef>& ProbeTraceDataArray,
        const TInlineArray<FRDGBufferRef>& HardwareRayTracingRayAllocatorBufferArray,
        const TInlineArray<FRDGBufferRef>& TraceProbesIndirectArgsArray,
        ERDGPassFlags ComputePassFlags);
}
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（RT-BLAS・TLAS へのアクセスに使用）|
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | Surface Cache アトラスバッファ（ヒット時のサンプル先）|
| `InputArray` | `const TInlineArray<FUpdateInputs>&` | 各 Radiance Cache の入力設定配列 |
| `OutputArray` | `TInlineArray<FUpdateOutputs>&` | 各 Radiance Cache の出力結果配列 |
| `SetupOutputArray` | `const TInlineArray<FRadianceCacheSetup>&` | 各 Radiance Cache の一時セットアップデータ（アトラステクスチャ等）|
| `ProbeTraceTileAllocatorArray` | `const TInlineArray<FRDGBufferRef>&` | 各キャッシュのトレースタイルカウンタバッファ配列 |
| `ProbeTraceTileDataArray` | `const TInlineArray<FRDGBufferRef>&` | 各キャッシュのトレースタイルデータバッファ配列 |
| `ProbeTraceDataArray` | `const TInlineArray<FRDGBufferRef>&` | 各プローブのワールド座標・インデックスバッファ配列 |
| `HardwareRayTracingRayAllocatorBufferArray` | `const TInlineArray<FRDGBufferRef>&` | Inline RT 用のレイバッファ動的割り当てカウンタ配列 |
| `TraceProbesIndirectArgsArray` | `const TInlineArray<FRDGBufferRef>&` | Indirect DispatchRays の引数バッファ配列 |
| `ComputePassFlags` | `ERDGPassFlags` | AsyncCompute / Compute の選択フラグ |

### 内部処理フロー

1. **RT パイプラインのセットアップ**
   ```cpp
   // FLumenHardwareRayTracingShaderBase から派生したシェーダーを選択
   // UseHitLighting() に応じて FullMaterial / LumenMinimal ペイロードを切り替え
   auto RayGenShader = GetLumenRadianceCacheHardwareRayTracingShader(View, Configuration);
   ```

2. **各 Radiance Cache に対してループ処理**
   ```cpp
   for (int32 CacheIndex = 0; CacheIndex < InputArray.Num(); ++CacheIndex) {
       // ProbeTraceTileAllocator / ProbeTraceTileData をシェーダーにバインド
       // ProbeTraceData（プローブのワールド座標・クリップマップ番号）をバインド
   }
   ```

3. **RT DispatchRays — プローブのトレース**
   ```cpp
   // TraceProbesIndirectArgs で Indirect Dispatch
   // タイルごとに 8×8 = 64 方向のレイを発射（TRACE_TILE_SIZE_2D = 8）
   // ペイロード: LumenMinimal（ヒット有無 + 座標のみ）
   GraphBuilder.AddPass(RDG_EVENT_NAME("RT RadianceCache"), PassParameters,
       [RayGenShader, ...](FRHIRayTracingCommandList& RHICmdList) {
           RHICmdList.RayTraceDispatchIndirect(..., TraceProbesIndirectArgs);
       });
   ```

4. **ヒット結果の書き込み**
   ```cpp
   // ヒット時: Surface Cache の FinalLightingAtlas をサンプリング → RadianceProbeAtlasTexture に書き込み
   // ミス時: スカイライト SH から Radiance を取得
   // UseHitLighting=true の場合: ヒット点でマテリアル評価を行い直接 Radiance を計算
   ```

5. **Far Field トレース（bFarField=true の場合）**
   ```cpp
   // 近距離 TLAS のミス後に Far Field TLAS に対して 2 段目のトレース
   // → 遠景オブジェクトからの Radiance を補完
   if (Configuration.bFarField) {
       GraphBuilder.AddPass(RDG_EVENT_NAME("RT RadianceCache FarField"), ...);
   }
   ```

### 使用箇所

- [[ref_lumen_radiance_cache]] — `UpdateRadianceCaches()` 内で `UsesHardwareRayTracing()` が true の場合に呼ばれる

---

## RenderLumenHardwareRayTracingRadianceCache_REMOVE（旧 API）

> [!note]- RenderLumenHardwareRayTracingRadianceCache_REMOVE — 単一キャッシュ旧 API（削除予定）
>
> ```cpp
> namespace LumenRadianceCache {
>     void RenderLumenHardwareRayTracingRadianceCache_REMOVE(
>         FRDGBuilder& GraphBuilder,
>         const FScene* Scene,
>         const FSceneTextureParameters& SceneTextures,
>         const FViewInfo& View,
>         const FLumenCardTracingParameters& TracingParameters,
>         const LumenRadianceCache::FRadianceCacheInterpolationParameters& RadianceCacheParameters,
>         FRadianceCacheConfiguration Configuration,
>         int32 MaxNumProbes,
>         int32 MaxProbeTraceTileResolution,
>         FRDGBufferRef ProbeTraceData,
>         FRDGBufferRef ProbeTraceTileData,
>         FRDGBufferRef ProbeTraceTileAllocator,
>         FRDGBufferRef TraceProbesIndirectArgs,
>         FRDGBufferRef HardwareRayTracingRayAllocatorBuffer,
>         FRDGBufferRef RadianceCacheHardwareRayTracingIndirectArgs,
>         FRDGTextureUAVRef RadianceProbeAtlasTextureUAV,
>         FRDGTextureUAVRef DepthProbeTextureUAV,
>         ERDGPassFlags ComputePassFlags);
> }
> ```
>
> 単一キャッシュを処理する旧 API。`_REMOVE` サフィックスが削除予定であることを示す。  
> 新しいコードでは複数キャッシュを同時処理できる `RenderLumenHardwareRayTracingRadianceCache()` を使用する。
>
> **パラメータの主な違い**
>
> | 項目 | 新 API | 旧 API |
> |------|--------|--------|
> | キャッシュ数 | `TInlineArray` で複数同時 | 単一キャッシュのみ |
> | 出力 | `RadianceCacheSetup` 経由 | `UAV` を直接受け取り |
> | アトラス | `FRadianceCacheSetup` から参照 | `RadianceProbeAtlasTextureUAV` を直接受け取り |
>
> **使用箇所**: 旧コードパスのみ。新規使用禁止

---

## トレーストークン（ProbeTraceTile）

```
ProbeTraceTileAllocator: トレースするタイル数カウンタ（Atomic カウンタ）
  → UpdateRadianceCaches() 内のプローブ選択パスが書き込む
  → TraceProbesIndirectArgs の生成に使用

ProbeTraceTileData: 各タイルのデータ（1 エントリ = 1 タイル = 64 方向）
  → 各エントリに { ProbeIndex, TileXY } を格納
  → Dispatch サイズ = タイル数 × 64 スレッド

ProbeTraceData:
  → 各プローブの { WorldPosition, ClipmapIndex, AtlasProbeIndex }
  → ProbeIndex を介して ProbeTraceTileData から参照

HardwareRayTracingRayAllocatorBuffer:
  → Inline RT モード（RHI_RAYTRACING_INLINE）時のレイバッファカウンタ
  → 1 スレッドで複数レイを順次発射するためのアロケータ
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
| Far Field | Surface Cache が届く範囲のみ | Far Field TLAS で遠景もカバー |
