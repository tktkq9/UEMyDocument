# リファレンス：LumenViewState.h

- グループ: Common
- 上位: [[02_lumen_overview]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenViewState.h`

---

## 概要

Lumen のビューごとの**テンポラル履歴（フレーム間蓄積）データ**を保持するクラス群。  
`FSceneViewState` に `FLumenViewState` として埋め込まれ、ビューの生存期間中保持される。

---

## クラス一覧

| クラス名 | 役割 |
|---------|------|
| `FLumenGatherCvarState` | Diffuse GI の CVar スナップショット（変化検知用）|
| `FScreenProbeGatherTemporalState` | Screen Probe GI のテンポラル履歴 |
| `FReSTIRTemporalResamplingState` | ReSTIR 時間方向リサンプリングの Reservoir 履歴 |
| `FReSTIRTemporalAccumulationState` | ReSTIR 時間方向蓄積の履歴 |
| `FReSTIRGatherTemporalState` | ReSTIR 全テンポラル状態のコンテナ |
| `FReflectionTemporalState` | 反射のテンポラル履歴 |
| `FRadianceCacheClipmap` | Radiance Cache の 1 クリップマップ段 |
| `FRadianceCacheState` | Radiance Cache の全 GPU リソース |
| `FLumenViewState` | 上記全ての頂点コンテナ |
| `FLumenCardPassUniformParameters` | Card キャプチャパスのUniform Buffer |

---

## FLumenGatherCvarState

CVar の変更を検知してテンポラル履歴を無効化するためのスナップショット。

```cpp
class FLumenGatherCvarState {
    int32 TraceMeshSDFs;         // r.Lumen.DiffuseIndirect.TraceMeshSDFs
    float MeshSDFTraceDistance;  // r.Lumen.DiffuseIndirect.MeshSDFTraceDistance
    float SurfaceBias;           // r.Lumen.DiffuseIndirect.SurfaceBias
    int32 VoxelTracingMode;      // r.Lumen.DiffuseIndirect.VoxelTracingMode
    int32 DirectLighting;        // 直接光設定

    bool operator==(const FLumenGatherCvarState& Rhs) const;
    // 全フィールドが一致するか比較 → 不一致なら履歴を SafeRelease
};
```

---

## FScreenProbeGatherTemporalState

Diffuse GI の Screen Probe テンポラル蓄積バッファ。  
詳細: [[e_lumen_diffuse_gi]]

| メンバ | 型 | 説明 |
|-------|-----|------|
| `DiffuseIndirectHistoryRT` | `IPooledRenderTarget` | Diffuse GI の前フレーム輝度 |
| `BackfaceDiffuseIndirectHistoryRT` | `IPooledRenderTarget` | 裏面の GI 履歴 |
| `RoughSpecularIndirectHistoryRT` | `IPooledRenderTarget` | ラフスペキュラー履歴 |
| `FastUpdateMode_NumFramesAccumulatedHistoryRT` | `IPooledRenderTarget` | 高速モードの蓄積フレーム数 |
| `ShortRangeAOHistoryRT` | `IPooledRenderTarget` | 短距離 AO 履歴 |
| `ShortRangeGIHistoryRT` | `IPooledRenderTarget` | 短距離 GI 履歴 |
| `HistoryScreenProbeSceneDepth` | `IPooledRenderTarget` | 前フレームの Probe 深度 |
| `HistoryScreenProbeTranslatedWorldPosition` | `IPooledRenderTarget` | 前フレームの Probe ワールド位置 |
| `ProbeHistoryScreenProbeRadiance` | `IPooledRenderTarget` | Probe 輝度履歴 |
| `ImportanceSamplingHistoryScreenProbeRadiance` | `IPooledRenderTarget` | Importance Sampling 用輝度履歴 |
| `LumenGatherCvars` | `FLumenGatherCvarState` | 設定変化検知スナップショット |
| `HistoryEffectiveResolution` | `FIntPoint` | 履歴の有効解像度 |

**主要メソッド:**

| メソッド | 説明 |
|---------|------|
| `SafeRelease()` | 全テクスチャを解放（設定変化時・初期化時に呼ばれる）|
| `AddCrossGPUTransfers(...)` | mGPU 環境での GPU 間転送登録（`WITH_MGPU` 時のみ）|
| `GetGPUSizeBytes(bool)` | GPU メモリ使用量を返す |

---

## FReSTIRTemporalResamplingState

ReSTIR の時間方向 Reservoir バッファ。詳細: [[f_lumen_reflections]]

| メンバ | 説明 |
|-------|------|
| `TemporalReservoirRayDirectionRT` | 各ピクセルの Reservoir レイ方向 |
| `TemporalReservoirTraceRadianceRT` | Reservoir のトレース輝度 |
| `TemporalReservoirTraceHitDistanceRT` | ヒット距離 |
| `TemporalReservoirTraceHitNormalRT` | ヒット法線 |
| `TemporalReservoirWeightsRT` | Reservoir の重み |
| `DownsampledDepthHistoryRT` | 前フレームのダウンサンプル深度 |
| `DownsampledNormalHistoryRT` | 前フレームのダウンサンプル法線 |
| `HistoryViewRect` | 前フレームのビュー矩形 |
| `HistoryReservoirViewSize` | Reservoir バッファのビューサイズ |

---

## FReflectionTemporalState

反射のテンポラル蓄積バッファ。詳細: [[f_lumen_reflections]]

| メンバ | 説明 |
|-------|------|
| `SpecularAndSecondMomentHistory` | 反射輝度 + 2次モーメント（分散計算用）|
| `NumFramesAccumulatedHistory` | 蓄積フレーム数 |
| `LayerSceneDepthHistory` | 前面透明レイヤーの深度履歴 |
| `LayerSceneNormalHistory` | 前面透明レイヤーの法線履歴 |
| `HistoryFrameIndex` | フレームインデックス（ジッター用）|
| `HistoryViewRect` | 前フレームのビュー矩形 |
| `HistoryScreenPositionScaleBias` | 再投影用スケール・バイアス |
| `HistoryUVMinMax` | UV の有効範囲 |
| `HistoryBufferSizeAndInvSize` | バッファサイズと逆数 |

---

## FRadianceCacheClipmap

Radiance Cache の 1 段分の状態。詳細: [[d_lumen_radiance_cache]]

| メンバ | 型 | 説明 |
|-------|-----|------|
| `Center` | `FVector` | このクリップマップのワールド空間中心 |
| `Extent` | `float` | クリップマップの半辺長 |
| `CornerWorldSpace` | `FVector3d` | グリッドコーナー位置（倍精度）|
| `CornerTranslatedWorldSpace` | `FVector3f` | 翻訳済みワールド座標 |
| `ProbeTMin` | `float` | レイの最小距離（自己交差防止）|
| `VolumeUVOffset` | `FVector` | スクロール更新用 UV オフセット |
| `CellSize` | `float` | プローブ間隔（ワールド単位）|

---

## FRadianceCacheState

Radiance Cache の GPU リソース全体。詳細: [[d_lumen_radiance_cache]]

| メンバ | 説明 |
|-------|------|
| `Clipmaps` | クリップマップ配列 |
| `ClipmapWorldExtent` | 全クリップマップの最大半径 |
| `ClipmapDistributionBase` | 指数的拡大の底 |
| `CachedLightingPreExposure` | キャッシュされた露出値 |
| `RadianceProbeIndirectionTexture` | Probe インデックス引き 3D テクスチャ |
| `RadianceProbeAtlasTexture` | Probe 放射輝度アトラス |
| `SkyVisibilityProbeAtlasTexture` | 空の可視性プローブアトラス |
| `FinalRadianceAtlas` | フィルタ済みラジアンスアトラス（参照用）|
| `FinalIrradianceAtlas` | 放射照度（積分済み）アトラス |
| `ProbeOcclusionAtlas` | プローブオクルージョンアトラス |
| `DepthProbeAtlasTexture` | プローブ深度アトラス |
| `ProbeAllocator` | 使用中プローブ数バッファ |
| `ProbeFreeListAllocator` | 空きリストサイズバッファ |
| `ProbeFreeList` | 再利用可能プローブインデックスリスト |
| `ProbeLastUsedFrame` | 最後に参照されたフレーム番号 |
| `ProbeLastTracedFrame` | 最後にトレースしたフレーム番号 |
| `ProbeWorldOffset` | プローブのワールドオフセット |

**主要メソッド:**

| メソッド | 説明 |
|---------|------|
| `ReleaseTextures()` | 全 GPU テクスチャを解放 |
| `AddCrossGPUTransfers(...)` | mGPU 環境での GPU 間転送登録 |
| `GetGPUSizeBytes(bool)` | GPU メモリ使用量を返す |

---

## FLumenViewState

ビューごとの Lumen テンポラル状態の頂点コンテナ。  
`FSceneViewState` に保持される。

```cpp
class FLumenViewState {
    // Diffuse GI テンポラル状態
    FScreenProbeGatherTemporalState ScreenProbeGatherState;
    FReSTIRGatherTemporalState      ReSTIRGatherState;

    // 反射テンポラル状態（不透明・半透明・水面の3種）
    FReflectionTemporalState ReflectionState;
    FReflectionTemporalState TranslucentReflectionState;
    FReflectionTemporalState WaterReflectionState;

    // 半透明ボリューム
    TRefCountPtr<IPooledRenderTarget> TranslucencyVolume0;
    TRefCountPtr<IPooledRenderTarget> TranslucencyVolume1;

    // Radiance Cache（不透明・半透明ボリュームの2種）
    FRadianceCacheState RadianceCacheState;
    FRadianceCacheState TranslucencyVolumeRadianceCacheState;

    void SafeRelease();
    void AddCrossGPUTransfers(...);  // WITH_MGPU
    uint64 GetGPUSizeBytes(bool) const;
};
```

---

## FLumenCardPassUniformParameters

Card キャプチャパス（Surface Cache への描画）で使う Global Uniform Buffer。

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FLumenCardPassUniformParameters, RENDERER_API)
    SHADER_PARAMETER_STRUCT(FSceneTextureUniformParameters, SceneTextures)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, EyeAdaptationBuffer)
    SHADER_PARAMETER(float, CachedLightingPreExposure)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

---

## 定数

```cpp
const static int32 NumLumenDiffuseIndirectTextures = 2;
const static int32 MaxVoxelClipmapLevels = 8;  // Radiance Cache 最大クリップマップ段数
```
