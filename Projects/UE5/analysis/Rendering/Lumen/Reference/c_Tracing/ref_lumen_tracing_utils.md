# リファレンス：LumenTracingUtils.h / LumenTracingUtils.cpp

- グループ: c - Tracing
- 上位: [[c_lumen_tracing]]
- 関連: [[ref_lumen_scene_data]] | [[ref_lumen_mesh_sdf_culling]] | [[ref_lumen_heightfields]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenTracingUtils.h/cpp`

---

## 概要

Lumen のトレーシング全体で共用されるシェーダーパラメータ構造体・ユーティリティ関数を定義するヘッダ。  
Surface Cache への Radiance サンプリング、Mesh SDF トレーシング、間接照明トレーシングのパラメータを一元管理する。

---

## FHemisphereDirectionSampleGenerator

半球（または全球）上のサンプル方向を事前生成するユーティリティクラス。  
Radiosity・Screen Probe Gather・Irradiance Field の全方向サンプリングに使われる。

```cpp
class FHemisphereDirectionSampleGenerator {
public:
    TArray<FVector4f> SampleDirections;  // 生成されたサンプル方向リスト
    float ConeHalfAngle = 0;             // 錐体トレース時の半角
    int32 Seed = 0;                      // 乱数シード
    int32 PowerOfTwoDivisor = 1;         // 2の冪乗での分割数
    bool bFullSphere = false;            // 全球サンプリングか（false=半球）
    bool bCosineDistribution = false;    // コサイン分布か（false=均一分布）

    void GenerateSamples(int32 TargetNumSamples, int32 InPowerOfTwoDivisor,
                         int32 InSeed, bool bInFullSphere, bool bInCosineDistribution);

    void GetSampleDirections(const FVector4f*& OutDirections, int32& OutNumDirections) const;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `SampleDirections` | `TArray<FVector4f>` | 生成されたサンプル方向（XYZ=方向, W=weight）|
| `ConeHalfAngle` | `float` | 各サンプルの錐体半角（ラジアン）。0 の場合は点サンプル |
| `Seed` | `int32` | 乱数シード。フレーム間でジッタを変化させる場合に使用 |
| `PowerOfTwoDivisor` | `int32` | テンポラルジッタ用の2の冪乗分割数。Radiosity では 4 など |
| `bFullSphere` | `bool` | true=全球, false=半球（デフォルト）|
| `bCosineDistribution` | `bool` | true=コサイン加重分布, false=均一球面分布 |

### 使用箇所

- [[ref_lumen_radiosity]] — `LumenRadiosity::InitFrameTemporaries()` でプローブ方向サンプル生成に使用
- [[ref_lumen_screen_probe_gather]] — Screen Probe の方向サンプル生成に使用
- [[ref_lumen_irradiance_field]] — `SetupRadianceCacheInputs()` のプローブレイ生成に使用

---

## FLumenCardTracingParameters

Surface Cache へのアクセスとフィードバック書き込みに必要な**全シェーダーパラメータ**。  
Lumen のほぼすべてのトレーシングシェーダーが `SHADER_PARAMETER_STRUCT_INCLUDE` でインクルードする。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenCardTracingParameters, )
    // ビュー・シーン
    SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneUniformParameters, Scene)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FReflectionUniformParameters, ReflectionStruct)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FLumenCardScene, LumenCardScene)

    // ライティング設定
    SHADER_PARAMETER(float, DiffuseColorBoost)
    SHADER_PARAMETER(float, SkylightLeakingRoughness)
    SHADER_PARAMETER(FVector3f, SkylightLeakingColor)
    SHADER_PARAMETER(float, ReflectionSkylightLeakingAverageAlbedo)
    SHADER_PARAMETER(float, InvFullSkylightLeakingDistance)
    SHADER_PARAMETER(float, CachedLightingPreExposure)
    SHADER_PARAMETER(float, OneOverCachedLightingPreExposure)

    // フォグ
    SHADER_PARAMETER(uint32, SampleHeightFog)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FFogUniformParameters, FogUniformParameters)

    // Surface Cache フィードバックバッファ（UAV）
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>,   RWCardPageLastUsedBuffer)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>,   RWCardPageHighResLastUsedBuffer)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>,   RWSurfaceCacheFeedbackBufferAllocator)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint2>,  RWSurfaceCacheFeedbackBuffer)
    SHADER_PARAMETER(uint32,    SurfaceCacheFeedbackBufferSize)
    SHADER_PARAMETER(uint32,    SurfaceCacheFeedbackBufferTileWrapMask)
    SHADER_PARAMETER(FIntPoint, SurfaceCacheFeedbackBufferTileJitter)
    SHADER_PARAMETER(float,     SurfaceCacheFeedbackResLevelBias)
    SHADER_PARAMETER(uint32,    SurfaceCacheUpdateFrameIndex)

    // Surface Cache アトラステクスチャ（SRV）
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DirectLightingAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, IndirectLightingAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, FinalLightingAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, AlbedoAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, OpacityAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, NormalAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, EmissiveAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DepthAtlas)

    // タイルシャドウ・Global SDF
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint4>, TileShadowDownsampleFactorAtlasForResampling)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, GlobalDistanceFieldPageObjectGridBuffer)
    SHADER_PARAMETER(uint32, NumGlobalSDFClipmaps)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `View` | `FViewUniformShaderParameters` | ビュー行列・投影行列・解像度等 |
| `Scene` | `FSceneUniformParameters` | シーン全体の共通パラメータ |
| `LumenCardScene` | `FLumenCardScene` | Card のアトラス UV マッピング情報 |
| `DiffuseColorBoost` | `float` | Diffuse Color の輝度ブースト係数 |
| `SkylightLeakingColor` | `FVector3f` | スカイライトリーク時の色（GBuffer の外側用）|
| `SkylightLeakingRoughness` | `float` | スカイライトリーク適用のラフネス閾値 |
| `InvFullSkylightLeakingDistance` | `float` | スカイライトリーク距離の逆数（フォールオフ計算用）|
| `CachedLightingPreExposure` | `float` | Surface Cache ライティングの Pre-Exposure 値 |
| `OneOverCachedLightingPreExposure` | `float` | PreExposure の逆数（復元用）|
| `SampleHeightFog` | `uint32` | ハイトフォグサンプリングの有効フラグ |
| `RWCardPageLastUsedBuffer` | `RWStructuredBuffer<uint>` | Card ページの最終使用フレーム書き込みバッファ |
| `RWCardPageHighResLastUsedBuffer` | `RWStructuredBuffer<uint>` | 高解像度ページの最終使用フレーム書き込みバッファ |
| `RWSurfaceCacheFeedbackBufferAllocator` | `RWStructuredBuffer<uint>` | フィードバックエントリのアロケータ（アトミックカウンタ）|
| `RWSurfaceCacheFeedbackBuffer` | `RWStructuredBuffer<uint2>` | フィードバックデータバッファ（CardPage ID + 要求解像度）|
| `SurfaceCacheFeedbackBufferSize` | `uint32` | フィードバックバッファのエントリ総数 |
| `SurfaceCacheFeedbackBufferTileWrapMask` | `uint32` | タイルインデックスのラップマスク |
| `SurfaceCacheFeedbackBufferTileJitter` | `FIntPoint` | フィードバックタイルのジッタオフセット |
| `SurfaceCacheFeedbackResLevelBias` | `float` | 要求解像度レベルのバイアス |
| `SurfaceCacheUpdateFrameIndex` | `uint32` | 現在のフレームインデックス（フィードバック識別用）|
| `DirectLightingAtlas` | `Texture2D` | 直接光アトラステクスチャ |
| `IndirectLightingAtlas` | `Texture2D` | 間接光アトラステクスチャ |
| `FinalLightingAtlas` | `Texture2D` | 最終合成ライティングアトラス |
| `AlbedoAtlas` | `Texture2D` | Albedo アトラス |
| `OpacityAtlas` | `Texture2D` | 不透明度アトラス |
| `NormalAtlas` | `Texture2D` | 法線アトラス |
| `EmissiveAtlas` | `Texture2D` | エミッシブアトラス |
| `DepthAtlas` | `Texture2D` | 深度アトラス |
| `NumGlobalSDFClipmaps` | `uint32` | Global SDF クリップマップ数 |

### 使用箇所

- [[ref_lumen_hwrt_common]] — `FLumenHardwareRayTracingShaderBase::FSharedParameters` に `INCLUDE` で組み込まれる
- [[ref_lumen_screen_probe_gather]] — Screen Probe トレースシェーダー全般
- [[ref_lumen_diffuse_indirect]] — `RenderLumenDiffuseIndirect()` 内部で初期化
- [[ref_lumen_reflections]] — Reflection トレースシェーダー全般

---

## GetLumenCardTracingParameters

```cpp
void GetLumenCardTracingParameters(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FLumenSceneData& LumenSceneData,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    bool bSurfaceCacheFeedback,
    FLumenCardTracingParameters& OutParameters);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `View` | `const FViewInfo&` | 現在のビュー |
| `LumenSceneData` | `const FLumenSceneData&` | Lumen シーンデータ |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | フレームのアトラスバッファ参照 |
| `bSurfaceCacheFeedback` | `bool` | Surface Cache フィードバックを書き込むか |
| `OutParameters` | `FLumenCardTracingParameters&` | 出力パラメータ構造体 |

### 内部処理フロー

1. **アトラステクスチャのバインド**
   ```cpp
   OutParameters.DirectLightingAtlas  = FrameTemporaries.DirectLightingAtlas;
   OutParameters.IndirectLightingAtlas = FrameTemporaries.IndirectLightingAtlas;
   OutParameters.FinalLightingAtlas   = FrameTemporaries.FinalLightingAtlas;
   OutParameters.AlbedoAtlas          = FrameTemporaries.AlbedoAtlas;
   // ... NormalAtlas, EmissiveAtlas, DepthAtlas
   ```

2. **フィードバックバッファのセットアップ**
   ```cpp
   if (bSurfaceCacheFeedback) {
       // FeedbackBuffer の UAV を取得して OutParameters にセット
       LumenSceneData.SurfaceCacheFeedback.GetShaderParameters(View, GraphBuilder,
           OutParameters.RWCardPageLastUsedBuffer,
           OutParameters.RWSurfaceCacheFeedbackBuffer, ...);
   } else {
       // ダミー UAV をセット（フィードバック無効時）
   }
   ```

3. **ライティング設定のコピー**
   ```cpp
   OutParameters.DiffuseColorBoost = FMath::Max(GDiffuseColorBoost, 1.0f);
   OutParameters.SkylightLeakingColor   = View.LumenSceneData->SkylightLeakingColor;
   OutParameters.CachedLightingPreExposure = View.PreExposure;
   ```

4. **フォグパラメータのバインド**
   ```cpp
   OutParameters.SampleHeightFog = ShouldRenderFog(ViewFamily) ? 1 : 0;
   if (OutParameters.SampleHeightFog) {
       OutParameters.FogUniformParameters = CreateFogUniformBuffer(GraphBuilder, View);
   }
   ```

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — `FScreenProbeGatherCS::FParameters` のセットアップ前に呼ばれる
- [[ref_lumen_reflections]] — `FLumenReflectionTracingCS` パラメータ構築時

---

## FLumenMeshSDFTracingParameters

Mesh SDF（符号付き距離フィールド）でのトレーシングに必要なパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenMeshSDFTracingParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FDistanceFieldObjectBufferParameters, DistanceFieldObjectBuffers)
    SHADER_PARAMETER_STRUCT_INCLUDE(FDistanceFieldAtlasParameters, DistanceFieldAtlas)
    SHADER_PARAMETER(float, MeshSDFNotCoveredExpandSurfaceScale)
    SHADER_PARAMETER(float, MeshSDFNotCoveredMinStepScale)
    SHADER_PARAMETER(float, MeshSDFDitheredTransparencyStepThreshold)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `DistanceFieldObjectBuffers` | `FDistanceFieldObjectBufferParameters` | SDF オブジェクトバッファ（位置・範囲・SDF インデックス）|
| `DistanceFieldAtlas` | `FDistanceFieldAtlasParameters` | SDF テクスチャアトラスへの参照 |
| `MeshSDFNotCoveredExpandSurfaceScale` | `float` | 両面マテリアル用サーフェス拡張スケール（デフォルト 0.6）|
| `MeshSDFNotCoveredMinStepScale` | `float` | 最小ステップサイズスケール（デフォルト 32、薄いジオメトリ高速スキップ用）|
| `MeshSDFDitheredTransparencyStepThreshold` | `float` | ディザ透明度の確率的スレッショルド（デフォルト 0.1）|

### 使用箇所

- [[ref_lumen_mesh_sdf_culling]] — `SetupLumenMeshSDFTracingParameters()` で生成される
- [[ref_lumen_screen_probe_tracing]] — Screen Probe SDF トレースシェーダーに渡される
- [[ref_lumen_diffuse_indirect]] — Mesh SDF GI パスのパラメータとして使用

---

## FLumenMeshSDFGridParameters

フラスタムグリッドに Mesh SDF オブジェクトをカリングした結果を持つパラメータ。  
`CullMeshObjectsToViewGrid()` の出力として使用される。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenMeshSDFGridParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FLumenMeshSDFTracingParameters, TracingParameters)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, NumGridCulledMeshSDFObjects)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, GridCulledMeshSDFObjectStartOffsetArray)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, GridCulledMeshSDFObjectIndicesArray)
    SHADER_PARAMETER(uint32,    CardGridPixelSizeShift)
    SHADER_PARAMETER(FVector3f, CardGridZParams)
    SHADER_PARAMETER(FIntVector, CullGridSize)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, NumCulledHeightfieldObjects)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, CulledHeightfieldObjectIndexBuffer)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, NumGridCulledHeightfieldObjects)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, GridCulledHeightfieldObjectStartOffsetArray)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, GridCulledHeightfieldObjectIndicesArray)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `TracingParameters` | `FLumenMeshSDFTracingParameters` | Mesh SDF トレーシング基本パラメータ |
| `NumGridCulledMeshSDFObjects` | `Buffer<uint>` | グリッドセルごとのカリング後 SDF オブジェクト数 |
| `GridCulledMeshSDFObjectStartOffsetArray` | `Buffer<uint>` | セルの開始オフセット配列（prefix sum）|
| `GridCulledMeshSDFObjectIndicesArray` | `Buffer<uint>` | セルに属するオブジェクトのインデックス配列 |
| `CardGridPixelSizeShift` | `uint32` | グリッドセルのピクセルサイズ（ビットシフト値、例: 6 = 64px）|
| `CardGridZParams` | `FVector3f` | Z スライスの深度マッピングパラメータ |
| `CullGridSize` | `FIntVector` | グリッドサイズ（X × Y × Z セル数）|
| `NumCulledHeightfieldObjects` | `Buffer<uint>` | ビューカリング後の Heightfield 総数 |
| `CulledHeightfieldObjectIndexBuffer` | `Buffer<uint>` | ビューカリング後の Heightfield インデックスバッファ |
| `NumGridCulledHeightfieldObjects` | `Buffer<uint>` | グリッドセルごとの Heightfield 数 |
| `GridCulledHeightfieldObjectStartOffsetArray` | `Buffer<uint>` | Heightfield セル開始オフセット |
| `GridCulledHeightfieldObjectIndicesArray` | `Buffer<uint>` | セルに属する Heightfield インデックス |

### 使用箇所

- [[ref_lumen_mesh_sdf_culling]] — `CullMeshObjectsToViewGrid()` の出力
- [[ref_lumen_screen_probe_tracing]] — SDF + Heightfield 統合トレースパスに渡される

---

## FLumenIndirectTracingParameters

間接照明トレーシングの距離・角度・バイアスを制御するパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenIndirectTracingParameters, )
    SHADER_PARAMETER(float, StepFactor)
    SHADER_PARAMETER(float, CardTraceEndDistanceFromCamera)
    SHADER_PARAMETER(float, DiffuseConeHalfAngle)
    SHADER_PARAMETER(float, TanDiffuseConeHalfAngle)
    SHADER_PARAMETER(float, MinSampleRadius)
    SHADER_PARAMETER(float, MinTraceDistance)
    SHADER_PARAMETER(float, MaxTraceDistance)
    SHADER_PARAMETER(float, MaxMeshSDFTraceDistance)
    SHADER_PARAMETER(float, SurfaceBias)
    SHADER_PARAMETER(float, CardInterpolateInfluenceRadius)
    SHADER_PARAMETER(float, SpecularFromDiffuseRoughnessStart)
    SHADER_PARAMETER(float, SpecularFromDiffuseRoughnessEnd)
    SHADER_PARAMETER(int32, HeightfieldMaxTracingSteps)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `StepFactor` | `float` | SDF レイマーチの進行係数（0<x≤1, 小さいほど精密）|
| `CardTraceEndDistanceFromCamera` | `float` | Surface Cache トレースが有効な最大カメラ距離（cm）|
| `DiffuseConeHalfAngle` | `float` | 拡散コーンの半角（ラジアン）|
| `TanDiffuseConeHalfAngle` | `float` | `tan(DiffuseConeHalfAngle)` の事前計算値 |
| `MinSampleRadius` | `float` | Surface Cache サンプルの最小半径（テクセル）|
| `MinTraceDistance` | `float` | レイ原点から最初のステップまでの距離（自己交差防止）|
| `MaxTraceDistance` | `float` | レイの最大トレース距離（cm）|
| `MaxMeshSDFTraceDistance` | `float` | Mesh SDF に切り替える最大距離（これ以遠は Global SDF）|
| `SurfaceBias` | `float` | サーフェスからのオフセットバイアス（法線方向）|
| `CardInterpolateInfluenceRadius` | `float` | Surface Cache 補間の影響半径 |
| `SpecularFromDiffuseRoughnessStart` | `float` | 鏡面→拡散切り替え開始ラフネス |
| `SpecularFromDiffuseRoughnessEnd` | `float` | 鏡面→拡散切り替え終了ラフネス |
| `HeightfieldMaxTracingSteps` | `int32` | Heightfield レイマーチの最大ステップ数 |

### 使用箇所

- [[ref_lumen_mesh_sdf_culling]] — `CullForCardTracing()` がこの構造体を受け取り、距離パラメータでカリング範囲を決定
- [[ref_lumen_screen_probe_tracing]] — Screen Probe 各トレースパスのパラメータとして使用
- [[ref_lumen_diffuse_indirect]] — `SetupLumenDiffuseTracingParameters()` で生成

---

## FLumenHZBScreenTraceParameters

HZB（Hierarchical Z Buffer）を使ったスクリーンスペーストレースのパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenHZBScreenTraceParameters, )
    SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture2D, PrevSceneColorTexture)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, HistorySceneDepth)
    SHADER_PARAMETER_STRUCT_INCLUDE(FHZBParameters, HZBParameters)
    SHADER_PARAMETER(FVector2f, PrevSceneColorBilinearUVMin)
    SHADER_PARAMETER(FVector2f, PrevSceneColorBilinearUVMax)
    SHADER_PARAMETER(FVector4f, PrevScreenPositionScaleBias)
    SHADER_PARAMETER(FVector4f, PrevScreenPositionScaleBiasForDepth)
    SHADER_PARAMETER(float, PrevSceneColorPreExposureCorrection)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `PrevSceneColorTexture` | `Texture2D` SRV | 前フレームのシーンカラー（スクリーントレースのヒット色取得用）|
| `HistorySceneDepth` | `Texture2D` | 前フレームの深度テクスチャ |
| `HZBParameters` | `FHZBParameters` | HZB ミップチェーンへのアクセスパラメータ |
| `PrevSceneColorBilinearUVMin/Max` | `FVector2f` | 前フレームカラーのサンプリング UV クランプ範囲 |
| `PrevScreenPositionScaleBias` | `FVector4f` | 前フレームスクリーン位置のスケール・バイアス変換 |
| `PrevScreenPositionScaleBiasForDepth` | `FVector4f` | 深度再投影用のスケール・バイアス |
| `PrevSceneColorPreExposureCorrection` | `float` | 前フレームの Pre-Exposure 補正係数 |

### 使用箇所

- [[ref_lumen_screen_probe_tracing]] — HZB スクリーントレースパスで使用
- [[ref_lumen_reflections]] — Reflection HZB スクリーントレースで使用

---

## FLumenScreenSpaceBentNormalParameters

Short Range AO / Screen Space Bent Normal を参照するパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenScreenSpaceBentNormalParameters, )
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<UNORM float>,  ShortRangeAOTexture)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float3>,       ShortRangeGITexture)
    SHADER_PARAMETER(FIntPoint, ShortRangeAOViewMin)
    SHADER_PARAMETER(FIntPoint, ShortRangeAOViewSize)
    SHADER_PARAMETER(uint32,    ShortRangeAOMode)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `ShortRangeAOTexture` | `Texture2D<UNORM float>` | Short Range AO 値テクスチャ（0=完全閉塞, 1=開放）|
| `ShortRangeGITexture` | `Texture2D<float3>` | スクリーンスペース GI テクスチャ（Bent Normal 方向）|
| `ShortRangeAOViewMin` | `FIntPoint` | AO テクスチャのビューオフセット（ピクセル）|
| `ShortRangeAOViewSize` | `FIntPoint` | AO テクスチャのビューサイズ（ピクセル）|
| `ShortRangeAOMode` | `uint32` | AO モード（0=無効, 1=AO のみ, 2=AO+Bent Normal）|

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — Screen Probe Gather の Denoising フェーズで AO マスクとして使用
- [[ref_lumen_short_range_ao]] — `RenderLumenShortRangeAO()` の出力をここに格納

---

## 主要ユーティリティ関数

| 関数 | 説明 |
|-----|------|
| `GetLumenCardTracingParameters(...)` | FLumenCardTracingParameters を初期化 |
| `CullHeightfieldObjectsForView(...)` | Heightfield オブジェクトをビューに対してカリング |
| `CullMeshObjectsToViewGrid(...)` | Mesh SDF オブジェクトをフラスタムグリッドにカリング |
| `CullForCardTracing(...)` | Card トレーシング用にカリング（IndirectTracingParameters ベース）|
| `SetupLumenDiffuseTracingParameters(...)` | 拡散トレーシングパラメータを設定 |
| `SetupLumenDiffuseTracingParametersForProbe(...)` | プローブ用拡散トレーシングパラメータを設定 |
| `SetupLumenMeshSDFTracingParameters(...)` | Mesh SDF トレーシングパラメータを設定 |
| `SetupHZBScreenTraceParameters(...)` | HZB スクリーントレースパラメータを設定 |

> [!note]- SetupLumenDiffuseTracingParameters — 拡散トレース距離・角度パラメータ生成
>
> ```cpp
> void SetupLumenDiffuseTracingParameters(
>     const FViewInfo& View,
>     float MaxTraceDistance,
>     float OrthoMaxDimension,
>     FLumenIndirectTracingParameters& OutParameters);
> ```
>
> **パラメータ**
>
> | 引数 | 型 | 説明 |
> |------|-----|------|
> | `View` | `const FViewInfo&` | 現在のビュー（FOV・解像度取得用）|
> | `MaxTraceDistance` | `float` | トレース最大距離（cm）|
> | `OrthoMaxDimension` | `float` | 正投影ビュー時の最大次元（cm, 遠近法ビューでは 0）|
> | `OutParameters` | `FLumenIndirectTracingParameters&` | 出力パラメータ |
>
> **内部動作**: CVar `r.Lumen.DiffuseIndirect.StepFactor` などから値を読み取り、  
> FOV に応じた `DiffuseConeHalfAngle` を計算して OutParameters に書き込む。  
>
> **使用箇所**: [[ref_lumen_diffuse_indirect]] — `RenderLumenDiffuseIndirect()` でパラメータ初期化時

> [!note]- SetupLumenDiffuseTracingParametersForProbe — プローブ用（角度オーバーライド版）
>
> ```cpp
> void SetupLumenDiffuseTracingParametersForProbe(
>     const FViewInfo& View,
>     float MaxTraceDistance,
>     float OrthoMaxDimension,
>     FLumenIndirectTracingParameters& OutParameters,
>     float DiffuseConeHalfAngle);
> ```
>
> **パラメータ**: `SetupLumenDiffuseTracingParameters` と同じだが、最後の引数で  
> `DiffuseConeHalfAngle` を外部から指定できる（プローブはコーンサイズが固定のため）。
>
> **使用箇所**: [[ref_lumen_screen_probe_gather]] — Screen Probe のトレース方向設定時

> [!note]- SetupHZBScreenTraceParameters — HZB スクリーントレースパラメータ生成
>
> ```cpp
> FLumenHZBScreenTraceParameters SetupHZBScreenTraceParameters(
>     FRDGBuilder& GraphBuilder,
>     const FViewInfo& View,
>     const FSceneTextures& SceneTextures);
> ```
>
> 前フレームのカラー・深度・HZB テクスチャを収集して `FLumenHZBScreenTraceParameters` を返す。  
> `PrevScreenPositionScaleBias` は TAA ジッタ補正済みの座標変換行列から計算する。
>
> **使用箇所**: [[ref_lumen_screen_probe_tracing]] — HZB スクリーントレースパス前に呼ばれる

---

## LumenIrradianceFieldGather 名前空間

```cpp
extern int32 GLumenIrradianceFieldGather; // CVar値（実験的機能の有効フラグ）

namespace LumenIrradianceFieldGather {
    LumenRadianceCache::FRadianceCacheInputs SetupRadianceCacheInputs();
}
```

### 使用箇所

- [[ref_lumen_irradiance_field]] — `LumenIrradianceFieldGather.cpp` の実装本体
- [[ref_lumen_diffuse_indirect]] — Irradiance Field モード選択時に `GLumenIrradianceFieldGather` を参照

---

## LumenDiffuseIndirect 名前空間

```cpp
namespace LumenDiffuseIndirect {
    bool IsAllowed();
    bool UseAsyncCompute(const FViewFamilyInfo& ViewFamily, EDiffuseIndirectMethod DiffuseIndirectMethod);
}
```

### 使用箇所

- [[ref_lumen_diffuse_indirect]] — `RenderLumenDiffuseIndirect()` のエントリで最初に `IsAllowed()` をチェック
