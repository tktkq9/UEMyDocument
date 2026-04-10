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

> **概要**: CVar の変更を検知してテンポラル履歴を無効化するためのスナップショット。

```cpp
class FLumenGatherCvarState {
    int32 TraceMeshSDFs;
    float MeshSDFTraceDistance;
    float SurfaceBias;
    int32 VoxelTracingMode;
    int32 DirectLighting;

    bool operator==(const FLumenGatherCvarState& Rhs) const;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `TraceMeshSDFs` | `int32` | `r.Lumen.DiffuseIndirect.TraceMeshSDFs` の現在値スナップショット |
| `MeshSDFTraceDistance` | `float` | `r.Lumen.DiffuseIndirect.MeshSDFTraceDistance` の現在値 |
| `SurfaceBias` | `float` | `r.Lumen.DiffuseIndirect.SurfaceBias` の現在値 |
| `VoxelTracingMode` | `int32` | `r.Lumen.DiffuseIndirect.VoxelTracingMode` の現在値 |
| `DirectLighting` | `int32` | 直接光設定の現在値 |

### 使用箇所

- [[ref_lumen_view_state]] — `FScreenProbeGatherTemporalState::LumenGatherCvars` として格納
- [[ref_lumen_screen_probe_gather]] — フレーム開始時に現在値と比較し、不一致なら `SafeRelease()` でテンポラル履歴を破棄

---

## FScreenProbeGatherTemporalState

> **概要**: Diffuse GI の Screen Probe テンポラル蓄積バッファ群。詳細: [[e_lumen_diffuse_gi]]

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
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

### 主要メソッド

| メソッド | 説明 |
|---------|------|
| `SafeRelease()` | 全テクスチャを解放（設定変化時・初期化時に呼ばれる）|
| `AddCrossGPUTransfers(...)` | mGPU 環境での GPU 間転送登録（`WITH_MGPU` 時のみ）|
| `GetGPUSizeBytes(bool)` | GPU メモリ使用量を返す |

### 使用箇所

- [[ref_lumen_view_state]] — `FLumenViewState::ScreenProbeGatherState` として保持
- [[ref_lumen_screen_probe_gather]] — `FReflectionTemporalCS` がフレームごとに `DiffuseIndirectHistoryRT` と現フレームをブレンド

---

## FReSTIRTemporalResamplingState

> **概要**: ReSTIR の時間方向 Reservoir バッファ群。詳細: [[f_lumen_reflections]]

### メンバ変数

| 変数名 | 説明 |
|--------|------|
| `TemporalReservoirRayDirectionRT` | 各ピクセルの Reservoir レイ方向 |
| `TemporalReservoirTraceRadianceRT` | Reservoir のトレース輝度 |
| `TemporalReservoirTraceHitDistanceRT` | ヒット距離 |
| `TemporalReservoirTraceHitNormalRT` | ヒット法線 |
| `TemporalReservoirWeightsRT` | Reservoir の重み（W_sum / M）|
| `DownsampledDepthHistoryRT` | 前フレームのダウンサンプル深度 |
| `DownsampledNormalHistoryRT` | 前フレームのダウンサンプル法線 |
| `HistoryViewRect` | 前フレームのビュー矩形 |
| `HistoryReservoirViewSize` | Reservoir バッファのビューサイズ |

### 使用箇所

- [[ref_lumen_view_state]] — `FReSTIRGatherTemporalState` に内包
- [[ref_lumen_restir]] — `FReSTIRParameters::PreviousReservoir` の構築時に外部テクスチャとして登録

---

## FReSTIRGatherTemporalState

> **概要**: ReSTIR 全テンポラル状態のコンテナ。

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `ResamplingState` | `FReSTIRTemporalResamplingState` | 時間方向リサンプリングの Reservoir 履歴 |
| `AccumulationState` | `FReSTIRTemporalAccumulationState` | 時間方向蓄積の履歴 |

### 使用箇所

- [[ref_lumen_view_state]] — `FLumenViewState::ReSTIRGatherState` として保持

---

## FReflectionTemporalState

> **概要**: 反射のテンポラル蓄積バッファ群。詳細: [[f_lumen_reflections]]

### メンバ変数

| 変数名 | 説明 |
|--------|------|
| `SpecularAndSecondMomentHistory` | 反射輝度 + 2次モーメント（分散計算用）|
| `NumFramesAccumulatedHistory` | 蓄積フレーム数（テンポラル安定性の制御に使用）|
| `LayerSceneDepthHistory` | 前面透明レイヤーの深度履歴 |
| `LayerSceneNormalHistory` | 前面透明レイヤーの法線履歴 |
| `HistoryFrameIndex` | フレームインデックス（ジッター用）|
| `HistoryViewRect` | 前フレームのビュー矩形 |
| `HistoryScreenPositionScaleBias` | 再投影用スケール・バイアス |
| `HistoryUVMinMax` | UV の有効範囲 |
| `HistoryBufferSizeAndInvSize` | バッファサイズと逆数 |

### 使用箇所

- [[ref_lumen_view_state]] — `FLumenViewState` に `ReflectionState` / `TranslucentReflectionState` / `WaterReflectionState` の3インスタンスとして保持
- [[ref_lumen_reflection_tracing]] — `FReflectionTemporalCS` が `SpecularAndSecondMomentHistory` と現フレームをブレンド

---

## FRadianceCacheClipmap

> **概要**: Radiance Cache の 1 段分のクリップマップ状態。詳細: [[d_lumen_radiance_cache]]

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Center` | `FVector` | このクリップマップのワールド空間中心 |
| `Extent` | `float` | クリップマップの半辺長 |
| `CornerWorldSpace` | `FVector3d` | グリッドコーナー位置（倍精度）|
| `CornerTranslatedWorldSpace` | `FVector3f` | 翻訳済みワールド座標 |
| `ProbeTMin` | `float` | レイの最小距離（自己交差防止）|
| `VolumeUVOffset` | `FVector` | スクロール更新用 UV オフセット |
| `CellSize` | `float` | プローブ間隔（ワールド単位）|

### 使用箇所

- [[ref_lumen_view_state]] — `FRadianceCacheState::Clipmaps` の要素
- [[ref_lumen_radiance_cache_internal]] — 各クリップマップのプローブ配置計算に参照

---

## FRadianceCacheState

> **概要**: Radiance Cache の GPU リソース全体。詳細: [[d_lumen_radiance_cache]]

### メンバ変数

| 変数名 | 説明 |
|--------|------|
| `Clipmaps` | `FRadianceCacheClipmap` の配列（最大 `MaxVoxelClipmapLevels = 8` 段）|
| `ClipmapWorldExtent` | 全クリップマップの最大半径 |
| `ClipmapDistributionBase` | 指数的拡大の底（各段の半径比率）|
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

### 主要メソッド

| メソッド | 説明 |
|---------|------|
| `ReleaseTextures()` | 全 GPU テクスチャを解放 |
| `AddCrossGPUTransfers(...)` | mGPU 環境での GPU 間転送登録 |
| `GetGPUSizeBytes(bool)` | GPU メモリ使用量を返す |

### 使用箇所

- [[ref_lumen_view_state]] — `FLumenViewState::RadianceCacheState` / `TranslucencyVolumeRadianceCacheState` として保持
- [[ref_lumen_radiance_cache]] — `UpdateLumenRadianceCache()` でフレームごとにプローブを更新
- [[ref_lumen_radiance_cache_internal]] — Probe 割り当て・解放・トレースパスが各アトラスに書き込む

---

## FLumenViewState

> **概要**: ビューごとの Lumen テンポラル状態の頂点コンテナ。`FSceneViewState` に保持される。

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

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `ScreenProbeGatherState` | `FScreenProbeGatherTemporalState` | Screen Probe GI テンポラル履歴 |
| `ReSTIRGatherState` | `FReSTIRGatherTemporalState` | ReSTIR Reservoir 履歴 |
| `ReflectionState` | `FReflectionTemporalState` | 不透明サーフェスの反射テンポラル履歴 |
| `TranslucentReflectionState` | `FReflectionTemporalState` | 透明サーフェスの反射テンポラル履歴 |
| `WaterReflectionState` | `FReflectionTemporalState` | 水面の反射テンポラル履歴 |
| `TranslucencyVolume0/1` | `IPooledRenderTarget` | Translucency Volume の2バッファ（Ping-Pong）|
| `RadianceCacheState` | `FRadianceCacheState` | 不透明用 Radiance Cache GPU リソース |
| `TranslucencyVolumeRadianceCacheState` | `FRadianceCacheState` | 半透明ボリューム用 Radiance Cache GPU リソース |

### 使用箇所

- `FSceneViewState` — ビューのライフタイム中保持。`GetLumenViewState()` でアクセス
- [[ref_lumen_screen_probe_gather]] — `View.ViewState->Lumen.ScreenProbeGatherState` を参照
- [[ref_lumen_reflections]] — `View.ViewState->Lumen.ReflectionState` を参照
- [[ref_lumen_radiance_cache]] — `View.ViewState->Lumen.RadianceCacheState` を参照

---

## FLumenCardPassUniformParameters

> **概要**: Card キャプチャパス（Surface Cache への描画）で使う Global Uniform Buffer。

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FLumenCardPassUniformParameters, RENDERER_API)
    SHADER_PARAMETER_STRUCT(FSceneTextureUniformParameters, SceneTextures)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, EyeAdaptationBuffer)
    SHADER_PARAMETER(float, CachedLightingPreExposure)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `SceneTextures` | `FSceneTextureUniformParameters` | GBuffer 等のシーンテクスチャ群 |
| `EyeAdaptationBuffer` | `StructuredBuffer<float4>` | 自動露出バッファ（Card 描画時の露出補正）|
| `CachedLightingPreExposure` | `float` | キャッシュされた露出値（`FRadianceCacheState::CachedLightingPreExposure` と共有）|

### 使用箇所

- [[ref_lumen_scene_card_capture]] — `RenderLumenCardCaptures()` でカード描画パスに渡す Uniform Buffer として設定

---

## 定数

```cpp
const static int32 NumLumenDiffuseIndirectTextures = 2;
const static int32 MaxVoxelClipmapLevels = 8;  // Radiance Cache 最大クリップマップ段数
```

| 定数名 | 値 | 説明 |
|-------|-----|------|
| `NumLumenDiffuseIndirectTextures` | 2 | Diffuse Indirect テクスチャのバッファ数（Ping-Pong 用）|
| `MaxVoxelClipmapLevels` | 8 | Radiance Cache クリップマップの最大段数 |

### 使用箇所

- [[ref_lumen_radiance_cache_internal]] — `MaxVoxelClipmapLevels` でクリップマップ配列サイズの上限を設定
- [[ref_lumen_screen_probe_gather]] — `NumLumenDiffuseIndirectTextures` でバッファ切り替えを管理
