# リファレンス：LumenReflections.h / LumenReflections.cpp

- グループ: f - Reflections
- 上位: [[f_lumen_reflections]]
- 関連: [[ref_lumen_reflection_tracing]] | [[ref_lumen_reflection_hwrt]] | [[ref_lumen_radiance_cache]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenReflections.h/cpp`

---

## 概要

Lumen Reflections システムの公開 API と主要パラメータ構造体を定義するヘッダ・実装ファイル。  
反射レイのトレーシング設定・タイル分類・コンパクト化・合成パラメータを管理する。  
SW RT / HW RT の両方に対応し、Radiance Cache との連携も担う。

---

## LumenReflections 名前空間

### FCompositeParameters

反射の合成（最終結合）で使うパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FCompositeParameters, )
    SHADER_PARAMETER(float, MaxRoughnessToTrace)
    SHADER_PARAMETER(float, MaxRoughnessToTraceForFoliage)
    SHADER_PARAMETER(float, InvRoughnessFadeLength)
    SHADER_PARAMETER(float, ReflectionSmoothBias)
END_SHADER_PARAMETER_STRUCT()
```

#### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `MaxRoughnessToTrace` | `float` | 反射レイをトレースする最大ラフネス（これより粗いサーフェスは反射なし）|
| `MaxRoughnessToTraceForFoliage` | `float` | フォリッジ用の最大ラフネス（葉などの Two-Sided 向け）|
| `InvRoughnessFadeLength` | `float` | MaxRoughnessToTrace 付近でフェードアウトする幅の逆数 |
| `ReflectionSmoothBias` | `float` | スムース面（低ラフネス）へのバイアス（品質優先で低ラフネスに補正）|

#### 使用箇所

- [[ref_lumen_reflections]] — `FLumenReflectionTracingParameters::ReflectionsCompositeParameters` にインクルード
- [[ref_lumen_reflections]] — `SetupCompositeParameters()` で PostProcess Volume 設定から構築

---

### 判定・ユーティリティ関数

| 関数 | 戻り値 | 説明 |
|------|--------|------|
| `SetupCompositeParameters(View, ReflectionsMethod, OutParams)` | `void` | FCompositeParameters を PostProcess 設定から構築 |
| `UseAsyncCompute(ViewFamily, DiffuseMethod, ReflMethod)` | `bool` | AsyncCompute を使うか（両パスで調整）|
| `UseFarField(ViewFamily)` | `bool` | Far Field TLAS を反射に使うか |
| `UseHitLighting(View, DiffuseMethod)` | `bool` | HW RT Hit Lighting を使うか |
| `UseTranslucentRayTracing(View)` | `bool` | 透明マテリアルの RT を使うか |
| `IsHitLightingForceEnabled(View, DiffuseMethod)` | `bool` | Hit Lighting を強制するか（CVar で制御）|
| `UseSurfaceCacheFeedback()` | `bool` | Surface Cache フィードバックを使うか |
| `UseScreenTraces(View)` | `bool` | HZB スクリーンスペーストレースを使うか |
| `UseDistantScreenTraces(View, bFarField, bRadianceCache)` | `bool` | 遠距離スクリーントレースを使うか |
| `GetDistantScreenTraceStepOffsetBias()` | `float` | 遠距離トレースのステップオフセットバイアス |
| `UseRadianceCache()` | `bool` | Radiance Cache を反射に使うか（`r.Lumen.Reflections.RadianceCache`）|
| `UseRadianceCacheSkyVisibility()` | `bool` | Sky Visibility を Radiance Cache で計算するか |
| `UseRadianceCacheStochasticInterpolation()` | `bool` | 確率的プローブ補間を使うか |
| `GetSampleSceneColorDepthTreshold()` | `float` | シーンカラーサンプリングの深度閾値 |
| `GetSampleSceneColorNormalTreshold()` | `float` | シーンカラーサンプリングの法線閾値 |
| `GetMaxReflectionBounces(View)` | `uint32` | 最大反射バウンス数（`r.Lumen.Reflections.MaxBounces`）|
| `GetMaxRefractionBounces(View)` | `uint32` | 最大屈折バウンス数 |

#### 使用箇所

- [[ref_lumen_reflection_tracing]] — `TraceReflections()` がトレース前に各フラグを確認
- [[ref_lumen_reflection_hwrt]] — `RenderLumenHardwareRayTracingReflections()` で HW RT パスを選択
- `DeferredShadingRenderer.cpp` — `ShouldRenderLumenReflections()` で反射パスを追加するか判定

---

## FLumenReflectionTracingParameters

反射トレーシング全体で共有するシェーダーパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenReflectionTracingParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FLumenReflectionsVisualizeTracesParameters, VisualizeTracesParameters)
    SHADER_PARAMETER(FIntPoint, ReflectionDownsampleFactorXY)
    SHADER_PARAMETER(FIntPoint, ReflectionTracingViewMin)
    SHADER_PARAMETER(FIntPoint, ReflectionTracingViewSize)
    SHADER_PARAMETER(FIntPoint, ReflectionTracingBufferSize)
    SHADER_PARAMETER(FVector2f, ReflectionTracingBufferInvSize)
    SHADER_PARAMETER(float,  MaxRayIntensity)
    SHADER_PARAMETER(uint32, ReflectionPass)
    SHADER_PARAMETER(uint32, UseJitter)
    SHADER_PARAMETER(uint32, UseHighResSurface)
    SHADER_PARAMETER(uint32, MaxReflectionBounces)
    SHADER_PARAMETER(uint32, MaxRefractionBounces)
    SHADER_PARAMETER(uint32, ReflectionsStateFrameIndex)
    SHADER_PARAMETER(uint32, ReflectionsStateFrameIndexMod8)
    SHADER_PARAMETER(uint32, ReflectionsRayDirectionFrameIndex)
    SHADER_PARAMETER(float, NearFieldMaxTraceDistance)
    SHADER_PARAMETER(float, NearFieldMaxTraceDistanceDitherScale)
    SHADER_PARAMETER(float, NearFieldSceneRadius)
    SHADER_PARAMETER(float, FarFieldMaxTraceDistance)
    SHADER_PARAMETER_STRUCT_INCLUDE(LumenReflections::FCompositeParameters, ReflectionsCompositeParameters)
    SHADER_PARAMETER_TEXTURE(Texture2D, PreIntegratedGF)
    SHADER_PARAMETER_SAMPLER(SamplerState, PreIntegratedGFSampler)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float4>, RayBuffer)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<uint>, RayTraceDistance)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DownsampledDepth)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DownsampledClosureIndex)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, TraceHit)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, TraceRadiance)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, TraceMaterialId)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, TraceBookmark)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2DArray<float3>, RWTraceRadiance)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2DArray<float3>, RWTraceBackgroundVisibility)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2DArray<float>,  RWTraceHit)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2DArray<uint>,   RWTraceMaterialId)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2DArray<uint2>,  RWTraceBookmark)
    SHADER_PARAMETER_STRUCT_REF(FBlueNoise, BlueNoise)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `VisualizeTracesParameters` | `FLumenReflectionsVisualizeTracesParameters` | デバッグ可視化パラメータ |
| `ReflectionDownsampleFactorXY` | `FIntPoint` | トレース解像度ダウンサンプル係数（X/Y 方向）|
| `ReflectionTracingViewMin` | `FIntPoint` | トレース対象ビュー領域の左上座標 |
| `ReflectionTracingViewSize` | `FIntPoint` | トレース対象ビュー領域のサイズ |
| `ReflectionTracingBufferSize` | `FIntPoint` | トレースバッファ全体サイズ |
| `ReflectionTracingBufferInvSize` | `FVector2f` | バッファサイズの逆数（UV 計算用）|
| `MaxRayIntensity` | `float` | レイ輝度クランプ値（ファイアフライ対策）|
| `ReflectionPass` | `uint32` | パス番号（マルチバウンス時のバウンスカウンタ）|
| `UseJitter` | `uint32` | ブルーノイズジッターを使うか（bool として使用）|
| `UseHighResSurface` | `uint32` | 高解像度サーフェスデータを使うか（bool として使用）|
| `MaxReflectionBounces` | `uint32` | 最大反射バウンス数 |
| `MaxRefractionBounces` | `uint32` | 最大屈折バウンス数（透明マテリアル用）|
| `ReflectionsStateFrameIndex` | `uint32` | テンポラル用フレームインデックス |
| `ReflectionsStateFrameIndexMod8` | `uint32` | フレームインデックスの mod 8（8フレームジッターパターン用）|
| `ReflectionsRayDirectionFrameIndex` | `uint32` | レイ方向ジッターのフレームインデックス |
| `NearFieldMaxTraceDistance` | `float` | Near Field トレースの最大距離（cm）|
| `NearFieldMaxTraceDistanceDitherScale` | `float` | Near Field 最大距離のディザリングスケール |
| `NearFieldSceneRadius` | `float` | Near Field シーン半径（カメラ中心からの距離）|
| `FarFieldMaxTraceDistance` | `float` | Far Field トレースの最大距離（cm）|
| `ReflectionsCompositeParameters` | `FCompositeParameters` | ラフネスフェード・最大ラフネス設定 |
| `PreIntegratedGF` | `Texture2D` | GGX の事前積分 G・F テーブル（スプリットサム近似用）|
| `PreIntegratedGFSampler` | `SamplerState` | PreIntegratedGF 用サンプラ |
| `RayBuffer` | `Texture2D<float4>` | 各ピクセルの反射レイ方向（xyz）と GGX PDF（w）|
| `RayTraceDistance` | `Texture2D<uint>` | 反射レイのトレース済み距離 |
| `DownsampledDepth` | `Texture2D` | ダウンサンプル済みシーン深度 |
| `DownsampledClosureIndex` | `Texture2D` | ダウンサンプル済みクロージャインデックス |
| `TraceHit` / `TraceRadiance` | `Texture2D` | トレース結果（読み取り用 SRV）|
| `TraceMaterialId` / `TraceBookmark` | `Texture2D` | Hit Lighting 用マテリアル ID / ブックマーク |
| `RWTraceRadiance` / `RWTraceHit` | `RWTexture2DArray` | トレース結果（書き込み用 UAV）|
| `RWTraceBackgroundVisibility` | `RWTexture2DArray<float3>` | 背景可視性（スカイ / Far Field）|
| `RWTraceMaterialId` / `RWTraceBookmark` | `RWTexture2DArray` | Hit Lighting 用（書き込み UAV）|
| `BlueNoise` | `FBlueNoise` | ブルーノイズテクスチャ（ジッター用）|

### 使用箇所

- [[ref_lumen_reflection_tracing]] — `TraceReflections()` の全サブパスに渡される
- [[ref_lumen_reflection_hwrt]] — `RenderLumenHardwareRayTracingReflections()` に渡される

---

## FLumenReflectionTileParameters

タイルベースの反射処理用パラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenReflectionTileParameters, )
    SHADER_PARAMETER_RDG_TEXTURE(Texture2DArray<uint>, LumenTileBitmask)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, ReflectionClearTileData)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, ReflectionResolveTileData)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, ReflectionTracingTileData)
    RDG_BUFFER_ACCESS(ClearIndirectArgs, ERHIAccess::IndirectArgs)
    RDG_BUFFER_ACCESS(ResolveIndirectArgs, ERHIAccess::IndirectArgs)
    RDG_BUFFER_ACCESS(TracingIndirectArgs, ERHIAccess::IndirectArgs)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `LumenTileBitmask` | `Texture2DArray<uint>` | タイルごとの分類ビットマスク（どのパスが必要かを示す）|
| `ReflectionClearTileData` | `Buffer<uint>` | クリアが必要なタイルのリスト |
| `ReflectionResolveTileData` | `Buffer<uint>` | 解像度復元が必要なタイルのリスト |
| `ReflectionTracingTileData` | `Buffer<uint>` | トレースが必要なタイルのリスト |
| `ClearIndirectArgs` | `IndirectArgs` | クリアパスの Indirect Dispatch Args |
| `ResolveIndirectArgs` | `IndirectArgs` | 解像度復元パスの Indirect Dispatch Args |
| `TracingIndirectArgs` | `IndirectArgs` | トレースパスの Indirect Dispatch Args |

### 使用箇所

- [[ref_lumen_reflection_tracing]] — `ClassifyReflectionTiles()` が構築し、各トレースパスに渡す
- [[ref_lumen_reflection_hwrt]] — HW RT トレース前のタイル分類に使用

---

## FCompactedReflectionTraceParameters

コンパクト化されたトレースパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FCompactedReflectionTraceParameters, )
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, CompactedTraceTexelAllocator)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, CompactedTraceTexelData)
    RDG_BUFFER_ACCESS(IndirectArgs, ERHIAccess::IndirectArgs)
    RDG_BUFFER_ACCESS(RayTraceDispatchIndirectArgs, ...)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `CompactedTraceTexelAllocator` | `Buffer<uint>` | 有効トレース数のカウンタ（Atomic）|
| `CompactedTraceTexelData` | `Buffer<uint>` | 各エントリに { PixelCoord, LayerIndex } を格納 |
| `IndirectArgs` | `IndirectArgs` | Compute Dispatch 用 Indirect Args |
| `RayTraceDispatchIndirectArgs` | `IndirectArgs` | RT DispatchRays 用 Indirect Args |

### 使用箇所

- [[ref_lumen_reflection_tracing]] — Mesh SDF / Global SDF トレース前のコンパクト化
- [[ref_lumen_reflection_hwrt]] — HW RT Dispatch 前のコンパクト化

---

## ETraceCompactionMode / CompactTraces

```cpp
enum ETraceCompactionMode { Default, FarField, HitLighting, MAX };

FCompactedReflectionTraceParameters LumenReflections::CompactTraces(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FLumenCardTracingParameters& TracingParameters,
    const FLumenReflectionTracingParameters& ReflectionTracingParameters,
    const FLumenReflectionTileParameters& ReflectionTileParameters,
    bool bCullByDistanceFromCamera,
    float CompactionTracingEndDistanceFromCamera,
    float CompactionMaxTraceDistance,
    ERDGPassFlags ComputePassFlags,
    ETraceCompactionMode TraceCompactionMode,
    bool bSortByMaterial);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `bCullByDistanceFromCamera` | `bool` | カメラ距離でカリングするか |
| `CompactionTracingEndDistanceFromCamera` | `float` | カメラからのトレース最大距離（cm）|
| `CompactionMaxTraceDistance` | `float` | レイの最大トレース距離（cm）|
| `TraceCompactionMode` | `ETraceCompactionMode` | コンパクト化モード（Default/FarField/HitLighting）|
| `bSortByMaterial` | `bool` | マテリアル別ソートを行うか（HitLighting 時の効率化）|

### 使用箇所

- [[ref_lumen_reflection_tracing]] — Mesh SDF / Global SDF 前に `Default` モードで実行
- [[ref_lumen_reflection_hwrt]] — HW RT 前に `Default`、Far Field に `FarField`、Hit Lighting に `HitLighting` モードで実行

---

## マイナー関数

> [!note]- ShouldRenderLumenReflections — 反射レンダリングの要否判定
>
> ```cpp
> bool ShouldRenderLumenReflections(const FSceneViewFamily& ViewFamily);
> ```
>
> **戻り値**: `bool`
>
> **有効条件**:
> - `r.Lumen.Reflections.Allow != 0`
> - `EngineShowFlags.LumenReflections` が有効
> - `DoesPlatformSupportLumenGI(ShaderPlatform)`
>
> **使用箇所**: `DeferredShadingRenderer.cpp` — 反射パスを追加するか判定

> [!note]- RenderLumenReflections — Lumen 反射の総合エントリポイント
>
> ```cpp
> void RenderLumenReflections(
>     FRDGBuilder& GraphBuilder,
>     const FScene* Scene,
>     const FViewInfo& View,
>     const FSceneTextures& SceneTextures,
>     const FLumenSceneFrameTemporaries& FrameTemporaries,
>     const FLumenReflectionsConfig& ReflectionsConfig,
>     FLumenMeshSDFGridParameters& MeshSDFGridParameters,
>     ERDGPassFlags ComputePassFlags);
> ```
>
> SW RT / HW RT を選択して `TraceReflections()` または `RenderLumenHardwareRayTracingReflections()` を呼ぶ。  
> Radiance Cache が有効な場合は `UpdateRadianceCaches()` も並行実行。
>
> **使用箇所**: `DeferredShadingRenderer.cpp` — 不透明パス後に呼ばれる

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.Reflections.Allow` | 1 | Lumen Reflections の有効/無効 |
| `r.Lumen.Reflections.DownsampleFactor` | 1 | トレース解像度ダウンサンプル係数 |
| `r.Lumen.Reflections.DownsampleCheckerboard` | 0 | チェッカーボードダウンサンプルの使用 |
| `r.Lumen.Reflections.MaxRoughnessToTrace` | -1.0 | トレースする最大ラフネス（-1=PostProcess設定を使用）|
| `r.Lumen.Reflections.MaxRoughnessToTraceClamp` | 1.0 | スケーラビリティ側の最大ラフネス上限 |
| `r.Lumen.Reflections.MaxRoughnessToTraceForFoliage` | 0.2 | フォリッジ用最大ラフネス |
| `r.Lumen.Reflections.RoughnessFadeLength` | 0.1 | ラフネスフェード幅 |
| `r.Lumen.Reflections.GGXSamplingBias` | 0.1 | GGX サンプリングバイアス |
| `r.Lumen.Reflections.Temporal` | 1 | テンポラルフィルタの有効/無効 |
| `r.Lumen.Reflections.Temporal.MaxFramesAccumulated` | 12.0 | テンポラル最大蓄積フレーム数 |
| `r.Lumen.Reflections.TraceMeshSDFs` | 1 | Mesh SDF トレースの有効/無効 |
| `r.Lumen.Reflections.RadianceCache` | 0 | Radiance Cache の使用（粗面の反射レイを短縮）|
| `r.Lumen.Reflections.RadianceCache.MinRoughness` | 0.2 | Radiance Cache を使い始めるラフネス閾値 |
| `r.Lumen.Reflections.RadianceCache.MaxRoughness` | 0.35 | Radiance Cache に完全切り替えるラフネス閾値 |
| `r.Lumen.Reflections.RadianceCache.MinTraceDistance` | 1000.0 | Radiance Cache 切り替え前の最小トレース距離 |
| `r.Lumen.Reflections.RadianceCache.MaxTraceDistance` | 5000.0 | Radiance Cache 切り替え前の最大トレース距離 |
