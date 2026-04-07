# リファレンス：Lumen.h / Lumen.cpp

- グループ: Common
- 上位: [[02_lumen_overview]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/Lumen.h` / `Lumen.cpp`

---

## 概要

Lumen システム全体の**エントリポイント・グローバル設定・機能判定関数**を定義するコアヘッダ。  
他のすべての Lumen ファイルがこのヘッダをインクルードする。

---

## グローバル関数（Lumen:: 名前空間外）

| 関数シグネチャ | 役割 |
|-------------|------|
| `bool ShouldRenderLumenDiffuseGI(const FScene*, const FSceneView&, bool bSkipTracingDataCheck, bool bSkipProjectCheck)` | Diffuse GI を描画すべきか判定 |
| `bool ShouldRenderLumenReflections(const FSceneView&, bool bSkipTracingDataCheck, bool bSkipProjectCheck, bool bIncludeStandalone)` | 反射を描画すべきか判定 |
| `bool ShouldRenderLumenReflectionsWater(const FViewInfo&, ...)` | 水面反射を描画すべきか判定 |
| `bool ShouldRenderLumenDirectLighting(const FScene*, const FSceneView&)` | Direct Lighting を描画すべきか判定 |
| `bool ShouldRenderAOWithLumenGI()` | Lumen GI 使用時に SSAO も描画するか（通常 false）|
| `bool ShouldUseStereoLumenOptimizations()` | ステレオ（VR）用最適化を使うか |
| `double BoxSurfaceArea(FVector Extent)` | AABB の表面積を計算（Card サイズ判定に使用）|

---

## Lumen 名前空間

### 定数（`constexpr`）

| 定数名 | 値 | 意味 |
|-------|-----|------|
| `PhysicalPageSize` | 128 | 物理ページのテクセルサイズ |
| `VirtualPageSize` | 127 | 仮想ページサイズ（0.5テクセルのボーダー分引く）|
| `MinCardResolution` | 8 | Card の最小解像度（テクセル）|
| `MinResLevel` | 3 | 最小解像度レベル（2^3 = 8px）|
| `MaxResLevel` | 11 | 最大解像度レベル（2^11 = 2048px）|
| `SubAllocationResLevel` | 7 | サブアロケーション境界 (log2(128))  |
| `NumResLevels` | 9 | 解像度レベル総数（11-3+1）|
| `CardTileSize` | 8 | Card タイルのサイズ（テクセル）|
| `NumDistanceBuckets` | 16 | 距離バケット数（Global SDF クリップマップ用）|
| `MaxTraceDistance` | 0.5f * UE_OLD_WORLD_MAX | レイの最大トレース距離 |

### enum class ETracingPermutation

```cpp
enum class ETracingPermutation {
    Cards,            // Surface Cache のみでトレース
    VoxelsAfterCards, // Cards → ミスしたら Voxel SDF
    Voxels,           // Voxel SDF のみ
    MAX
};
```

### enum class ESurfaceCacheSampling

```cpp
enum class ESurfaceCacheSampling {
    AlwaysResidentPagesWithoutFeedback, // 常駐ページのみ・フィードバックなし
    AlwaysResidentPages,                // 常駐ページのみ
    HighResPages,                       // 高解像度ページ優先
};
```

### シーン・ビュー判定関数

| 関数 | 役割 |
|------|------|
| `IsLumenFeatureAllowedForView(Scene, View, ...)` | 指定ビューで Lumen 機能が有効か |
| `ShouldVisualizeScene(ShowFlags)` | デバッグ可視化モードか |
| `ShouldVisualizeHardwareRayTracing(ViewFamily)` | HW RT の可視化モードか |
| `ShouldHandleSkyLight(Scene, ViewFamily)` | スカイライトを Lumen で処理するか |
| `ShouldUpdateLumenSceneViewOrigin()` | Lumen シーンの視点原点を更新するか |
| `GetLumenSceneViewOrigin(View, ClipmapIndex)` | クリップマップ段別の視点原点を取得 |

### Global Distance Field 関連

| 関数 | 役割 |
|------|------|
| `GetGlobalDFResolution()` | Global SDF の解像度を返す |
| `GetGlobalDFClipmapExtent(ClipmapIndex)` | 指定段のクリップマップ半径を返す |
| `GetNumGlobalDFClipmaps(View)` | 使用するクリップマップ段数を返す |

### GPU 機能フラグ

| 関数 | 役割 |
|------|------|
| `UseAsyncCompute(ViewFamily)` | AsyncCompute を使うか |
| `UseWaveOps(ShaderPlatform)` | Wave Intrinsics を使うか |
| `UseThreadGroupSize32()` | スレッドグループサイズを 32 にするか（AMD RDNA 等）|
| `GetLightingDataFormat()` | ライティングデータのピクセルフォーマットを返す |
| `GetLightingQuantizationError()` | 量子化誤差ベクトルを返す |
| `GetCachedLightingPreExposure()` | キャッシュされた露出値を返す |

### Surface Cache 状態確認

| 関数 | 役割 |
|------|------|
| `IsSurfaceCacheFrozen()` | Surface Cache 更新が凍結中か |
| `IsSurfaceCacheUpdateFrameFrozen()` | 更新フレームが凍結中か |

### Software Ray Tracing（SDF）判定

| 関数 | 役割 |
|------|------|
| `IsSoftwareRayTracingSupported()` | SRT がサポートされているか |
| `UseMeshSDFTracing(ShowFlags)` | Mesh SDF トレースを使うか |
| `UseGlobalSDFTracing(ShowFlags)` | Global SDF トレースを使うか |
| `UseGlobalSDFSimpleCoverageBasedExpand()` | SDF 展開の簡易モードを使うか |
| `UseGlobalSDFObjectGrid(ViewFamily)` | SDF オブジェクトグリッドを使うか |
| `UseHeightfieldTracing(ViewFamily, SceneData)` | ハイトフィールドトレースを使うか |
| `UseHeightfieldTracingForVoxelLighting(SceneData)` | Voxel ライティング用 HF トレースを使うか |
| `GetHeightfieldMaxTracingSteps()` | ハイトフィールドの最大ステップ数 |
| `IsUsingGlobalSDF(ViewFamily)` | Global SDF を使っているか |
| `IsUsingDistanceFieldRepresentationBit(View)` | DFビットを使っているか |

### Hardware Ray Tracing 判定

| 関数 | 役割 |
|------|------|
| `AnyLumenHardwareRayTracingPassEnabled(Scene, View)` | いずれかの HW RT パスが有効か |
| `UseHardwareRayTracing(ViewFamily)` | HW RT を使うか（大元のスイッチ）|
| `UseHardwareRayTracedSceneLighting(ViewFamily)` | Scene Lighting に HW RT を使うか |
| `UseHardwareRayTracedDirectLighting(ViewFamily)` | Direct Lighting に HW RT を使うか |
| `UseHardwareRayTracedReflections(ViewFamily)` | 反射に HW RT を使うか |
| `UseReSTIRGather(ViewFamily, Platform)` | ReSTIR Gather を使うか |
| `UseHardwareRayTracedScreenProbeGather(ViewFamily)` | Screen Probe に HW RT を使うか |
| `UseHardwareRayTracedShortRangeAO(ViewFamily)` | 短距離 AO に HW RT を使うか |
| `UseHardwareRayTracedRadianceCache(ViewFamily)` | Radiance Cache に HW RT を使うか |
| `UseHardwareRayTracedRadiosity(ViewFamily)` | Radiosity に HW RT を使うか |
| `ShouldRenderRadiosityHardwareRayTracing(ViewFamily)` | Radiosity HW RT を描画するか |
| `IsUsingRayTracingLightingGrid(ViewFamily, View, Method)` | RT ライティンググリッドを使うか |
| `UseHardwareInlineRayTracing(ViewFamily)` | インライン RT を使うか（DXR 1.1 Inline）|

### Far Field / Near Field 設定

| 関数 | 役割 |
|------|------|
| `UseFarField(ViewFamily)` | 遠距離フィールドを使うか |
| `UseFarFieldOcclusionOnly()` | 遠距離はオクルージョンのみか |
| `GetFarFieldMaxTraceDistance()` | 遠距離の最大トレース距離 |
| `GetNearFieldMaxTraceDistanceDitherScale(bUseFarField)` | 近距離フェードのスケール |
| `GetNearFieldSceneRadius(View, bUseFarField)` | 近距離の有効半径 |

### その他ユーティリティ

| 関数 | 役割 |
|------|------|
| `GetMeshCardDistanceBin(Distance)` | 距離→バケットインデックス変換 |
| `GetHeightfieldReceiverBias()` | ハイトフィールド受光バイアス値 |
| `Shutdown()` | Lumen システムのシャットダウン |
| `WriteWarnings(Scene, ShowFlags, Views, Writer)` | デバッグ警告メッセージの出力 |
| `DebugResetSurfaceCache()` | Surface Cache を強制リセット（デバッグ用）|
| `GetMaxTraceDistance(View)` | ビューに応じた最大トレース距離 |
| `SupportsMultipleClosureEvaluation(Platform/View)` | Substrate 多重クロージャ評価対応か |

---

## グローバル変数

```cpp
extern int32 GLumenFastCameraMode;  // r.LumenScene.FastCameraMode の値
LLM_DECLARE_TAG(Lumen);             // LLM（Low Level Memory）タグ
```

---

## 主要 CVar（Lumen.cpp で定義）

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Lumen.Supported` | 1 | Lumen を有効にするか（ReadOnly）|
| `r.Lumen.Supported.SM5` | 0 | 旧 SM5 パスで Lumen を許可するか |
| `r.Lumen.AsyncCompute` | 1 | 非同期 Compute を使うか |
| `r.Lumen.WaveOps` | 1 | Wave Intrinsics を使うか |
| `r.Lumen.ThreadGroupSize32` | 1 | スレッドグループサイズを 32 にするか |
| `r.Lumen.LightingDataFormat` | 0 | 0=R11G11B10 / 1=Float16RGBA / 2=Float32RGBA |
