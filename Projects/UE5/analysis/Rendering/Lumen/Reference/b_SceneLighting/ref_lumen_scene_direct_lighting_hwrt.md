# リファレンス：LumenSceneDirectLightingHardwareRayTracing.cpp

- グループ: b - Scene Lighting
- 上位: [[b_lumen_scene_lighting]]
- 関連: [[ref_lumen_scene_direct_lighting]] | [[ref_lumen_hwrt_common]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneDirectLightingHardwareRayTracing.cpp`

---

## 概要

Surface Cache Direct Lighting の **Hardware Ray Tracing（HW RT）シャドウ**実装ファイル。  
SDF ベースのシャドウと比較して、より正確なシャドウを高解像度 Card に提供する。  
`#if RHI_RAYTRACING` ブロック内に実装されており、RT 対応 GPU でのみコンパイルされる。

---

## Lumen 名前空間（HW RT 判定）

```cpp
namespace Lumen {
    bool UseHardwareRayTracedDirectLighting(const FSceneViewFamily& ViewFamily);
}
```

### UseHardwareRayTracedDirectLighting

```cpp
bool Lumen::UseHardwareRayTracedDirectLighting(const FSceneViewFamily& ViewFamily) {
    return IsRayTracingEnabled()
        && Lumen::UseHardwareRayTracing(ViewFamily)
        && (CVarLumenSceneDirectLightingHardwareRayTracing.GetValueOnRenderThread() != 0);
}
```

### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `RenderDirectLightingStandard()` — HW RT / SDF の分岐判定
- [[ref_lumen_scene_lighting]] `FDeferredShadingRenderer::RenderLumenSceneLighting()` — 使用判定

---

## LumenSceneDirectLighting 名前空間（HW RT 関連）

```cpp
namespace LumenSceneDirectLighting {
    bool UseFarField(const FSceneViewFamily& ViewFamily);
    bool IsForceTwoSided();
}
```

### UseFarField

```cpp
bool LumenSceneDirectLighting::UseFarField(const FSceneViewFamily& ViewFamily) {
    return Lumen::UseFarField(ViewFamily)
        && (CVarLumenSceneDirectLightingHardwareRayTracingFarField.GetValueOnRenderThread() != 0);
}
```

### IsForceTwoSided

```cpp
bool LumenSceneDirectLighting::IsForceTwoSided() {
    return CVarLumenSceneDirectLightingHardwareRayTracingForceTwoSided.GetValueOnRenderThread() != 0;
}
```

---

## FLumenSceneDebugHardwareRayTracing

HW RT を使ったデバッグ用シャドウトレースシェーダー。  
`TraceLumenHardwareRayTracedDebug()` から呼ばれ、デバッグ描画に使用される。

```cpp
class FLumenSceneDebugHardwareRayTracing : public FLumenHardwareRayTracingShaderBase {
    DECLARE_LUMEN_RAYTRACING_SHADER(FLumenSceneDebugHardwareRayTracing)

    using FPermutationDomain = TShaderPermutationDomain<
        FLumenHardwareRayTracingShaderBase::FBasePermutationDomain>;

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_STRUCT_INCLUDE(
            FLumenHardwareRayTracingShaderBase::FSharedParameters, SharedParameters)
        SHADER_PARAMETER_STRUCT_INCLUDE(ShaderPrint::FShaderParameters, ShaderPrintUniformBuffer)
        SHADER_PARAMETER(float, ResolutionScale)
        SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>, RWDebugData)
    END_SHADER_PARAMETER_STRUCT()

    static ERayTracingPayloadType GetRayTracingPayloadType(const int32 PermutationId);
};
```

### メンバ変数（FParameters）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `SharedParameters` | `FSharedParameters` | HW RT 共通パラメータ（BLAS・シーン・ビューなど）|
| `ShaderPrintUniformBuffer` | `ShaderPrint::FShaderParameters` | ShaderPrint デバッグ出力用 UB |
| `ResolutionScale` | `float` | デバッグ表示の解像度スケール（= ViewRect.Width / UnscaledWidth）|
| `RWDebugData` | `RWStructuredBuffer<uint>` | デバッグデータ出力バッファ |

### GetRayTracingPayloadType

```cpp
static ERayTracingPayloadType GetRayTracingPayloadType(const int32 PermutationId) {
    return ERayTracingPayloadType::LumenMinimal; // 位置・法線のみ（マテリアル評価なし）
}
```

### 使用箇所
- [[ref_lumen_scene_direct_lighting_hwrt]] `TraceLumenHardwareRayTracedDebug()` のみから呼ばれる

---

## FLumenDirectLightingHardwareRayTracing

Direct Lighting シャドウ用の HW RT シェーダー（CS / RGS の両方を生成）。  
`IMPLEMENT_LUMEN_RAYGEN_AND_COMPUTE_RAYTRACING_SHADERS` マクロで両方登録される。

```cpp
class FLumenDirectLightingHardwareRayTracing : public FLumenHardwareRayTracingShaderBase {
    class FForceTwoSided                   : SHADER_PERMUTATION_BOOL("FORCE_TWO_SIDED");
    class FEnableFarFieldTracing           : SHADER_PERMUTATION_BOOL("FAR_FIELD_TRACING");
    class FEnableHeightfieldProjectionBias : SHADER_PERMUTATION_BOOL("HEIGHTFIELD_PROJECTION_BIAS");
    class FSurfaceCacheAlphaMasking        : SHADER_PERMUTATION_BOOL("ALPHA_MASKING");
    class FStochastic                      : SHADER_PERMUTATION_BOOL("STOCHASTIC");

    using FPermutationDomain = TShaderPermutationDomain<
        FLumenHardwareRayTracingShaderBase::FBasePermutationDomain,
        FForceTwoSided, FEnableFarFieldTracing,
        FEnableHeightfieldProjectionBias, FSurfaceCacheAlphaMasking,
        FStochastic>;
};
```

### パーミュテーション

| パーミュテーション | 説明 |
|--------------------|------|
| `FForceTwoSided` | 全メッシュを両面扱いにするか（`IsForceTwoSided()` の結果）|
| `FEnableFarFieldTracing` | Far Field トレースを有効にするか |
| `FEnableHeightfieldProjectionBias` | Heightfield 投影バイアスを適用するか |
| `FSurfaceCacheAlphaMasking` | Surface Cache アルファマスキングを有効にするか |
| `FStochastic` | Stochastic Lighting モード（Screen Probe 連携）を使うか |

### メンバ変数（FParameters）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `HardwareRayTracingIndirectArgs` | `FRDGBufferRef` | Dispatch Indirect 引数バッファ |
| `LightTileAllocator/LightTiles` | `StructuredBuffer<uint/uint2>` | ライトタイルバッファ（SRV）|
| `LumenLightData` | `FLumenLightDataParameters` | パックされたライトデータ |
| `ShadowTraceAllocator` | `SRV` | シャドウトレース数カウンタ |
| `ShadowTraces` | `SRV` | シャドウトレースリスト |
| `PullbackBias` | `float` | 自己遮蔽防止のプルバックバイアス（通常 0.0）|
| `ViewIndex` | `int32` | マルチビューインデックス |
| `MaxTraceDistance` | `float` | 最大トレース距離 |
| `FarFieldMaxTraceDistance` | `float` | Far Field の最大トレース距離 |
| `HardwareRayTracingShadowRayBias` | `float` | シャドウレイバイアス |
| `HardwareRayTracingEndBias` | `float` | プロキシ自己遮蔽防止のエンドバイアス |
| `HeightfieldShadowReceiverBias` | `float` | Heightfield シャドウ受信バイアス |
| `HeightfieldProjectionBiasSearchRadius` | `float` | Heightfield 投影バイアス探索半径 |
| `RWShadowMaskTiles` | `UAV` | 出力: シャドウマスクタイル（0=遮蔽, 1=光あり）|
| `ViewExposure` | `float` | ビュー露光値（Stochastic 用）|
| `FrustumTranslatedWorldToClip` | `FMatrix44f` | フラスタム行列（Stochastic 用）|
| `PreViewTranslation` | `FVector3f` | ビュー事前移動（Stochastic 用）|
| `RWLightSamples` | `UAV` | 出力: ライトサンプルテクスチャ（Stochastic 用）|
| `CompactedLightSampleData` | `SRV` | コンパクトライトサンプルデータ（Stochastic 用）|
| `CompactedLightSampleAllocator` | `SRV` | コンパクトライトサンプル数（Stochastic 用）|

### 使用箇所
- [[ref_lumen_scene_direct_lighting_hwrt]] `TraceLumenHardwareRayTracedDirectLightingShadows()` から実行

---

## FLumenDirectLightingHardwareRayTracingIndirectArgsCS

HW RT シャドウ用の Dispatch 間接引数を計算する CS。

```cpp
class FLumenDirectLightingHardwareRayTracingIndirectArgsCS : public FGlobalShader {
    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>, DispatchLightTilesIndirectArgs)
        SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>, RWHardwareRayTracingIndirectArgs)
        SHADER_PARAMETER(uint32, OutputThreadGroupSize)
        SHADER_PARAMETER(int32, bStochastic)
    END_SHADER_PARAMETER_STRUCT()
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `DispatchLightTilesIndirectArgs` | `SRV` | ライトタイル数から作った元 Indirect Args |
| `RWHardwareRayTracingIndirectArgs` | `UAV` | 出力: HW RT 用に変換した Indirect Args |
| `OutputThreadGroupSize` | `uint32` | 出力スレッドグループサイズ（64 固定）|
| `bStochastic` | `int32` | Stochastic モードか（引数の計算式が変わる）|

### 使用箇所
- [[ref_lumen_scene_direct_lighting_hwrt]] `TraceLumenHardwareRayTracedDirectLightingShadows()` — HW RT 実行前に呼ばれる

---

## エントリポイント関数

### TraceLumenHardwareRayTracedDirectLightingShadows

```cpp
void TraceLumenHardwareRayTracedDirectLightingShadows(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    const FLumenCardUpdateContext& CardUpdateContext,
    const FLumenCardTileUpdateContext& CardTileUpdateContext,
    const TArray<FLumenGatheredLight>& GatheredLights,
    FRDGBufferRef ShadowMaskTiles);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（BLAS・ライト情報参照）|
| `View` | `const FViewInfo&` | ビュー情報 |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | Card Scene UB など |
| `CardUpdateContext` | `const FLumenCardUpdateContext&` | 更新 Card ページコンテキスト |
| `CardTileUpdateContext` | `const FLumenCardTileUpdateContext&` | タイル単位コンテキスト |
| `GatheredLights` | `const TArray<FLumenGatheredLight>&` | シーン内ライト一覧 |
| `ShadowMaskTiles` | `FRDGBufferRef` | 出力: シャドウマスクバッファ（0/1）|

#### 使用箇所
- [[ref_lumen_scene_direct_lighting]] `RenderDirectLightingStandard()` — HW RT 有効時

#### 内部処理フロー

1. **Dispatch 間接引数の構築**
   ```cpp
   FRDGBufferRef HardwareRayTracingIndirectArgsBuffer =
       GraphBuilder.CreateBuffer(FRDGBufferDesc::CreateStructuredDesc(sizeof(uint32)*4, 1), ...);
   
   AddPass(FLumenDirectLightingHardwareRayTracingIndirectArgsCS,
       PassParams = { DispatchLightTilesIndirectArgs, RWHardwareRayTracingIndirectArgs,
                      OutputThreadGroupSize = 64, bStochastic });
   ```

2. **FLumenCardTracingParameters の取得**
   ```cpp
   FLumenCardTracingParameters TracingParams;
   GetLumenCardTracingParameters(View, FrameTemporaries, TracingParams);
   ```

3. **シェーダーパラメータの設定**
   ```cpp
   auto* PassParams = GraphBuilder.AllocParameters<FLumenDirectLightingHardwareRayTracing::FParameters>();
   SetLumenHardwareRayTracedDirectLightingShadowsParameters(PassParams, ...);
   
   // PullbackBias = 0.0 (Card は既に法線方向にオフセット済み)
   // MaxTraceDistance = Lumen::GetMaxTraceDistance()
   // HardwareRayTracingShadowRayBias = GetHardwareRayTracingShadowRayBias()
   // HardwareRayTracingEndBias = CVarEndBias (default: 1.0)
   ```

4. **Stochastic 用追加パラメータ（FStochastic = true 時）**
   ```cpp
   PassParams->FrustumTranslatedWorldToClip = View.ViewMatrices.GetTranslatedViewProjectionMatrix();
   PassParams->PreViewTranslation = View.ViewMatrices.GetPreViewTranslation();
   PassParams->RWLightSamples = GraphBuilder.CreateUAV(StochasticData.LightSamples);
   PassParams->CompactedLightSampleData = StochasticData.CompactedLightSampleData;
   PassParams->CompactedLightSampleAllocator = StochasticData.CompactedLightSampleAllocator;
   ```

5. **パーミュテーション設定と実行**
   ```cpp
   FPermutationDomain PermutationVector;
   PermutationVector.Set<FForceTwoSided>(IsForceTwoSided());
   PermutationVector.Set<FEnableFarFieldTracing>(UseFarField(ViewFamily));
   PermutationVector.Set<FStochastic>(UseStochasticLighting(ViewFamily));
   
   if (bInlineRayTracing) {
       FLumenDirectLightingHardwareRayTracingCS::AddLumenRayTracingDispatchIndirect(...);
   } else {
       FLumenDirectLightingHardwareRayTracingRGS::AddLumenRayTracingDispatchIndirect(...);
   }
   ```

---

### TraceLumenHardwareRayTracedDebug

```cpp
FRDGBufferSRVRef TraceLumenHardwareRayTracedDebug(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    float ResolutionScale);
```

#### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（BLAS 参照）|
| `View` | `const FViewInfo&` | ビュー情報 |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | Card Scene UB |
| `ResolutionScale` | `float` | デバッグ表示解像度スケール |

#### 戻り値
`FRDGBufferSRVRef` — デバッグデータバッファの SRV

#### 使用箇所
- [[ref_lumen_scene_rendering]] `RenderLumenScene()` — `r.Lumen.Visualize.HardwareRayTracing` が有効な場合

#### 内部処理フロー

```cpp
FRDGBufferSRVRef TraceLumenHardwareRayTracedDebug(...)
{
    // 出力バッファ作成
    FRDGBufferRef OutDebugBuffer = GraphBuilder.CreateBuffer(...);
    
    // パラメータ設定
    auto* PassParams = GraphBuilder.AllocParameters<FLumenSceneDebugHardwareRayTracing::FParameters>();
    SetLumenHardwareRayTracingSharedParameters(GraphBuilder, View, FrameTemporaries,
        &PassParams->SharedParameters);
    PassParams->ShaderPrintUniformBuffer = ShaderPrint::GetParameters(View);
    PassParams->ResolutionScale = ResolutionScale;
    PassParams->RWDebugData = GraphBuilder.CreateUAV(OutDebugBuffer);
    
    // 実行（CS or RGS）
    if (bInlineRayTracing) {
        FLumenSceneDebugHardwareRayTracingCS::AddLumenRayTracingDispatch(...);
    } else {
        FLumenSceneDebugHardwareRayTracingRGS::AddLumenRayTracingDispatch(...);
    }
    
    return GraphBuilder.CreateSRV(OutDebugBuffer);
}
```

---

> [!note]- GetHeightfieldProjectionBiasSearchRadius — 探索半径取得
> 
> ```cpp
> float GetHeightfieldProjectionBiasSearchRadius() {
>     return FMath::Max(
>         CVarLumenSceneDirectLightingHardwareRayTracingHeightfieldProjectionBiasSearchRadius.GetValueOnRenderThread(),
>         0.0f);
> }
> ```
> 
> **使用箇所**: [[ref_lumen_scene_direct_lighting_hwrt]] `SetLumenHardwareRayTracedDirectLightingShadowsParameters()` — HW RT パラメータ設定

---

> [!note]- PrepareLumenHardwareRayTracingDirectLightingLumenMaterial — 準備処理
> 
> ```cpp
> void FDeferredShadingSceneRenderer::PrepareLumenHardwareRayTracingDirectLightingLumenMaterial(
>     const FViewInfo& View,
>     TArray<FRHIRayTracingShader*>& OutRayGenShaders);
> ```
> 
> **説明**: 使用するパーミュテーションを事前に決定し、RayGenShader を登録する。
> 
> **内部動作**:
> ```cpp
> if (UseHardwareRayTracedDirectLighting && !UseHardwareInlineRayTracing) {
>     FPermutationDomain PermutationVector;
>     PermutationVector.Set<FForceTwoSided>(IsForceTwoSided());
>     PermutationVector.Set<FEnableFarFieldTracing>(UseFarField(ViewFamily));
>     // ... 他のパーミュテーション設定
>     auto RayGenShader = GetGlobalShaderMap(...)->GetShader<FLumenDirectLightingHardwareRayTracingRGS>(PermutationVector);
>     OutRayGenShaders.Add(RayGenShader.GetRayTracingShader());
> }
> ```
> 
> **使用箇所**: `FDeferredShadingSceneRenderer::PrepareRayTracingLumenHardwareRayTracing()` から呼ばれる

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.DirectLighting.HardwareRayTracing` | 1 | HW RT Direct Lighting の有効/無効 |
| `r.LumenScene.DirectLighting.HardwareRayTracing.ForceTwoSided` | 0 | 全メッシュを両面扱いにするか |
| `r.LumenScene.DirectLighting.HardwareRayTracing.EndBias` | 1.0 | プロキシジオメトリの自己遮蔽防止バイアス |
| `r.LumenScene.DirectLighting.HardwareRayTracing.FarField` | 1 | Far Field を HW RT シャドウで使うか |
| `r.LumenScene.DirectLighting.HardwareRayTracing.HeightfieldProjectionBias` | 0 | Heightfield 投影バイアスを適用するか |
| `r.LumenScene.DirectLighting.HardwareRayTracing.HeightfieldProjectionBiasSearchRadius` | 256 | 投影バイアスの探索半径（大きいほど負荷増）|
| `r.LumenScene.DirectLighting.HardwareRayTracing.AdaptiveShadowTracing` | 1 | 均一シャドウタイルのレイ数を削減するか |

---

## HW RT シャドウの仕組み

```
TraceLumenHardwareRayTracedDirectLightingShadows()
  │
  ├─ FLumenDirectLightingHardwareRayTracingIndirectArgsCS
  │   └─ ライトタイル数から HW RT 用 Dispatch 間接引数を計算
  │
  ├─ 各 Card タイル × 各ライトに対してシャドウレイを Dispatch
  │   ├─ ペイロード: ERayTracingPayloadType::LumenMinimal（位置・法線のみ → 高速）
  │   ├─ AdaptiveShadowTracing: 前フレームで均一シャドウ → レイ削減
  │   └─ ForceTwoSided: false → 裏面はシャドウを落とさない
  │
  ├─ Far Field 有効時: 近距離 BLAS の miss 後に Far Field をトレース
  │
  └─ RWShadowMaskTiles に 0(遮蔽) / 1(光あり) を書き込み
        → FLumenCardBatchDirectLightingCS で Direct Lighting アトラスに合成
```

---

## SDF シャドウとの比較

| 比較項目 | SDF シャドウ | HW RT シャドウ |
|---------|------------|--------------|
| 精度 | 低〜中（SDF の近似誤差）| 高（ジオメトリに対して正確）|
| 速度 | 速い | 遅い（RT 対応ハードが必要）|
| 有効条件 | 常に利用可 | RT 対応 GPU のみ |
| バイアス調整 | MeshSDF / GlobalSDF 別に設定 | HardwareRayTracing.ShadowRayBias |
| 両面設定 | MeshSDF では CullMode に従う | ForceTwoSided で上書き可能 |
| Far Field | Global SDF のみ | 専用の BLAS を使用 |
