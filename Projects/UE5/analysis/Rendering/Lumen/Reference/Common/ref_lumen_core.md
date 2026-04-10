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

### 使用箇所

- [[ref_lumen_diffuse_indirect]] — `ShouldRenderLumenDiffuseGI()` でフレーム冒頭の描画可否を判定
- [[ref_lumen_reflections]] — `ShouldRenderLumenReflections()` でフレーム冒頭の描画可否を判定
- [[ref_lumen_scene_lighting]] — `ShouldRenderLumenDirectLighting()` で直接光注入パスを制御
- [[ref_lumen_mesh_cards]] — `BoxSurfaceArea()` で Mesh Card のサイズ優先度を計算

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
| `SubAllocationResLevel` | 7 | サブアロケーション境界 (log2(128)) |
| `NumResLevels` | 9 | 解像度レベル総数（11-3+1）|
| `CardTileSize` | 8 | Card タイルのサイズ（テクセル）|
| `NumDistanceBuckets` | 16 | 距離バケット数（Global SDF クリップマップ用）|
| `MaxTraceDistance` | 0.5f * UE_OLD_WORLD_MAX | レイの最大トレース距離 |

### 使用箇所

- [[ref_lumen_scene_data]] — `PhysicalPageSize` / `VirtualPageSize` で Surface Cache ページ計算
- [[ref_lumen_mesh_cards]] — `MinResLevel` / `MaxResLevel` / `NumResLevels` でCard解像度レベル管理
- [[ref_lumen_tracing_utils]] — `MaxTraceDistance` でトレースの最大距離上限をクランプ

---

### enum class ETracingPermutation

```cpp
enum class ETracingPermutation {
    Cards,            // Surface Cache のみでトレース
    VoxelsAfterCards, // Cards → ミスしたら Voxel SDF
    Voxels,           // Voxel SDF のみ
    MAX
};
```

| 値 | 説明 |
|----|------|
| `Cards` | Surface Cache の Card アトラスのみを参照。高精度だが Card 未登録エリアはミス |
| `VoxelsAfterCards` | Cards でミスした後に Voxel SDF でカバー。標準モード |
| `Voxels` | Voxel SDF のみ。低品質だが広域カバー（デバッグ・遠距離用）|

### 使用箇所

- [[ref_lumen_screen_probe_tracing]] — Screen Probe トレース時に `ETracingPermutation` でシェーダーパーミュテーションを選択
- [[ref_lumen_tracing_utils]] — `FLumenCardTraceParameters` の構築時にパーミュテーションを設定

---

### enum class ESurfaceCacheSampling

```cpp
enum class ESurfaceCacheSampling {
    AlwaysResidentPagesWithoutFeedback, // 常駐ページのみ・フィードバックなし
    AlwaysResidentPages,                // 常駐ページのみ
    HighResPages,                       // 高解像度ページ優先
};
```

| 値 | 説明 |
|----|------|
| `AlwaysResidentPagesWithoutFeedback` | 常駐ページのみ参照・フィードバックループなし（Shadow など副産物トレース用）|
| `AlwaysResidentPages` | 常駐ページのみ参照・フィードバックあり（Screen Probe 等）|
| `HighResPages` | 高解像度ページを優先参照（反射など品質重視パス）|

### 使用箇所

- [[ref_lumen_surface_cache_feedback]] — フィードバックループで `ESurfaceCacheSampling` 別に優先度を設定
- [[ref_lumen_reflections]] — 反射パスは `HighResPages` で高解像度 Card を優先参照

---

### シーン・ビュー判定関数

| 関数 | 役割 |
|------|------|
| `IsLumenFeatureAllowedForView(Scene, View, ...)` | 指定ビューで Lumen 機能が有効か |
| `ShouldVisualizeScene(ShowFlags)` | デバッグ可視化モードか |
| `ShouldVisualizeHardwareRayTracing(ViewFamily)` | HW RT の可視化モードか |
| `ShouldHandleSkyLight(Scene, ViewFamily)` | スカイライトを Lumen で処理するか |
| `ShouldUpdateLumenSceneViewOrigin()` | Lumen シーンの視点原点を更新するか |
| `GetLumenSceneViewOrigin(View, ClipmapIndex)` | クリップマップ段別の視点原点を取得 |

### 使用箇所

- [[ref_lumen_scene]] — `IsLumenFeatureAllowedForView()` でビューごとに Lumen 更新を制御
- [[ref_lumen_visualize]] — `ShouldVisualizeScene()` / `ShouldVisualizeHardwareRayTracing()` で可視化パスの追加を判定
- [[ref_lumen_radiance_cache]] — `GetLumenSceneViewOrigin()` でクリップマップの中心座標を取得

---

### Global Distance Field 関連

| 関数 | 役割 |
|------|------|
| `GetGlobalDFResolution()` | Global SDF の解像度を返す |
| `GetGlobalDFClipmapExtent(ClipmapIndex)` | 指定段のクリップマップ半径を返す |
| `GetNumGlobalDFClipmaps(View)` | 使用するクリップマップ段数を返す |

### 使用箇所

- [[ref_lumen_tracing_utils]] — Global SDF トレース時にクリップマップ解像度・段数を取得
- [[ref_lumen_screen_probe_tracing]] — Global SDF フォールバックパスで `GetNumGlobalDFClipmaps()` を使用

---

### GPU 機能フラグ

| 関数 | 役割 |
|------|------|
| `UseAsyncCompute(ViewFamily)` | AsyncCompute を使うか |
| `UseWaveOps(ShaderPlatform)` | Wave Intrinsics を使うか |
| `UseThreadGroupSize32()` | スレッドグループサイズを 32 にするか（AMD RDNA 等）|
| `GetLightingDataFormat()` | ライティングデータのピクセルフォーマットを返す |
| `GetLightingQuantizationError()` | 量子化誤差ベクトルを返す |
| `GetCachedLightingPreExposure()` | キャッシュされた露出値を返す |

### 使用箇所

- [[ref_lumen_scene_lighting]] — `UseAsyncCompute()` で照明パスを AsyncCompute キューに投入するか決定
- [[ref_lumen_screen_probe_gather]] — `UseWaveOps()` でシェーダーパーミュテーションを選択
- [[ref_lumen_scene_data]] — `GetLightingDataFormat()` で `FinalLightingAtlas` のフォーマット指定

---

### Surface Cache 状態確認

| 関数 | 役割 |
|------|------|
| `IsSurfaceCacheFrozen()` | Surface Cache 更新が凍結中か |
| `IsSurfaceCacheUpdateFrameFrozen()` | 更新フレームが凍結中か |

### 使用箇所

- [[ref_lumen_surface_cache]] — `IsSurfaceCacheFrozen()` で `UpdateLumenSurfaceCache()` 内の更新処理をスキップするか判定

---

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

### 使用箇所

- [[ref_lumen_reflection_tracing]] — `UseMeshSDFTracing()` / `UseGlobalSDFTracing()` でトレースパスの有効化を制御
- [[ref_lumen_heightfields]] — `UseHeightfieldTracing()` でハイトフィールドパスを制御
- [[ref_lumen_mesh_sdf_culling]] — `IsSoftwareRayTracingSupported()` でカリングパスの有効化を確認

---

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

### 使用箇所

- [[ref_lumen_reflections]] — `UseHardwareRayTracedReflections()` で SW / HW RT の分岐を制御
- [[ref_lumen_screen_probe_hwrt]] — `UseHardwareRayTracedScreenProbeGather()` で Screen Probe HW RT パスを制御
- [[ref_lumen_restir]] — `UseReSTIRGather()` で ReSTIR パスの有効化を判定
- [[ref_lumen_hwrt_common]] — `UseHardwareInlineRayTracing()` で Inline RT / RGS の選択に使用

---

### Far Field / Near Field 設定

| 関数 | 役割 |
|------|------|
| `UseFarField(ViewFamily)` | 遠距離フィールドを使うか |
| `UseFarFieldOcclusionOnly()` | 遠距離はオクルージョンのみか |
| `GetFarFieldMaxTraceDistance()` | 遠距離の最大トレース距離 |
| `GetNearFieldMaxTraceDistanceDitherScale(bUseFarField)` | 近距離フェードのスケール |
| `GetNearFieldSceneRadius(View, bUseFarField)` | 近距離の有効半径 |

### 使用箇所

- [[ref_lumen_reflection_tracing]] — `UseFarField()` で Far Field コンパクト化パスを制御
- [[ref_lumen_reflection_hwrt]] — `GetFarFieldMaxTraceDistance()` で Far Field TLAS トレース距離を設定
- [[ref_lumen_tracing_utils]] — `GetNearFieldSceneRadius()` で Near Field コンパクト化の距離上限を決定

---

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

### 使用箇所

- [[ref_lumen_scene_gpu_driven_update]] — `GetMeshCardDistanceBin()` でカードの距離バケット分類に使用
- [[ref_lumen_scene_rendering]] — `WriteWarnings()` でデバッグ情報をビューポートに表示
- [[ref_lumen_hwrt_materials]] — `SupportsMultipleClosureEvaluation()` で Substrate マルチクロージャ評価を有効化

---

## グローバル変数

```cpp
extern int32 GLumenFastCameraMode;  // r.LumenScene.FastCameraMode の値
LLM_DECLARE_TAG(Lumen);             // LLM（Low Level Memory）タグ
```

### 使用箇所

- [[ref_lumen_scene]] — `GLumenFastCameraMode` でカメラ高速移動時の品質削減モードを制御

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
