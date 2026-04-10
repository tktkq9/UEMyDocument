# リファレンス：LumenHardwareRayTracingCommon.h / LumenHardwareRayTracingCommon.cpp

- グループ: c - Tracing
- 上位: [[c_lumen_tracing]]
- 関連: [[ref_lumen_tracing_utils]] | [[ref_lumen_hwrt_materials]] | [[ref_lumen_scene_direct_lighting_hwrt]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenHardwareRayTracingCommon.h/cpp`

---

## 概要

Lumen の **Hardware Ray Tracing（HW RT）** 全機能で共有される基底クラス・マクロ・ユーティリティを定義する。  
GI・Reflections・Direct Lighting・Radiosity など、すべての Lumen RT シェーダーがこのファイルの  
`FLumenHardwareRayTracingShaderBase` を継承する。

---

## LumenHardwareRayTracing 名前空間

### 列挙型

```cpp
namespace LumenHardwareRayTracing {

    // 自己交差回避モード
    enum class EAvoidSelfIntersectionsMode : uint8 {
        Disabled, // 無効（高速）
        Retrace,  // 自己交差時にリトレース
        AHS,      // Any Hit Shader で回避
        MAX
    };

    // ヒット時のライティングモード
    enum class EHitLightingMode {
        SurfaceCache,              // Surface Cache からサンプリング（デフォルト）
        HitLighting,               // ヒット点でフルマテリアル評価（GI+Reflection）
        HitLightingForReflections, // ヒット点でフルマテリアル評価（Reflectionのみ）
        MAX
    };
}
```

### ユーティリティ関数

```cpp
namespace LumenHardwareRayTracing {
    bool IsInlineSupported();
    bool IsRayGenSupported();
    float GetFarFieldBias();

    bool UseSurfaceCacheAlphaMasking();
    EAvoidSelfIntersectionsMode GetAvoidSelfIntersectionsMode();

    EHitLightingMode GetHitLightingMode(const FViewInfo& View,
        EDiffuseIndirectMethod DiffuseIndirectMethod);
    uint32 GetHitLightingShadowMode();
    uint32 GetHitLightingShadowTranslucencyMode();
    bool UseHitLightingForceOpaque();
    bool UseHitLightingDirectLighting();
    bool UseHitLightingSkylight(EDiffuseIndirectMethod);
    bool UseReflectionCapturesForHitLighting();
    bool UseShaderExecutionReordering();

    void SetRayTracingSceneOptions(const FViewInfo& View,
        EDiffuseIndirectMethod DiffuseIndirectMethod,
        EReflectionsMethod ReflectionsMethod,
        RayTracing::FSceneOptions& SceneOptions);
}
```

| 関数 | 戻り値 | 説明 |
|------|--------|------|
| `IsInlineSupported()` | `bool` | Inline RT（CS 内トレース）対応プラットフォームか |
| `IsRayGenSupported()` | `bool` | RayGen シェーダー対応プラットフォームか |
| `GetFarFieldBias()` | `float` | Far Field トレースのバイアス値 |
| `GetAvoidSelfIntersectionsMode()` | `EAvoidSelfIntersectionsMode` | 現在の自己交差回避モード（CVar から取得）|
| `GetHitLightingMode()` | `EHitLightingMode` | ビューと DiffuseIndirectMethod から HitLighting モードを決定 |
| `GetHitLightingShadowMode()` | `uint32` | Hit Lighting のシャドウモード（0=無効,1=ハード,2=ソフト）|
| `UseShaderExecutionReordering()` | `bool` | SER（Shader Execution Reordering）の使用可否 |
| `SetRayTracingSceneOptions()` | `void` | RT シーンビルド時のオプションを CVar から構築 |

### 使用箇所

- [[ref_lumen_scene_direct_lighting_hwrt]] — `GetAvoidSelfIntersectionsMode()` でシェーダーパーミュテーション選択
- [[ref_lumen_reflections]] — `GetHitLightingMode()` で Reflection の HitLighting 判定
- [[ref_lumen_hwrt_materials]] — `NumHitGroups` 計算に `EAvoidSelfIntersectionsMode` を使用

---

## FLumenHardwareRayTracingShaderBase

すべての Lumen RT シェーダーの**基底クラス**。`FGlobalShader` を継承。  
TLAS・Nanite RT・ライトグリッド・Surface Cache パラメータをまとめて管理する。

```cpp
class FLumenHardwareRayTracingShaderBase : public FGlobalShader {
public:
    BEGIN_SHADER_PARAMETER_STRUCT(FSharedParameters, )
        // シーンテクスチャ
        SHADER_PARAMETER_STRUCT_INCLUDE(FSceneTextureParameters, SceneTextures)
        SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneTextureUniformParameters, SceneTexturesStruct)
        SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSubstrateGlobalUniformParameters, Substrate)

        // RT アクセラレーション構造
        SHADER_PARAMETER_RDG_BUFFER_SRV(RaytracingAccelerationStructure, TLAS)
        SHADER_PARAMETER_RDG_BUFFER_SRV(RaytracingAccelerationStructure, FarFieldTLAS)
        SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer, RayTracingSceneMetadata)
        SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>, RWInstanceHitCountBuffer)

        // Nanite
        SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FNaniteRayTracingUniformParameters, NaniteRayTracing)

        // ライティング
        SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FRayTracingLightGrid, LightGridParameters)
        SHADER_PARAMETER_STRUCT_REF(FReflectionCaptureShaderData, ReflectionCapture)
        SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FForwardLightUniformParameters, ForwardLightStruct)

        // Lumen Surface Cache
        SHADER_PARAMETER_STRUCT_INCLUDE(FLumenCardTracingParameters, TracingParameters)
        SHADER_PARAMETER(uint32, MaxTraversalIterations)
        SHADER_PARAMETER(uint32, MeshSectionVisibilityTest)
        SHADER_PARAMETER(float, MinTraceDistanceToSampleSurfaceCache)
        SHADER_PARAMETER(float, SurfaceCacheSamplingDepthBias)

        // Inline RT 用
        SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<Lumen::FHitGroupRootConstants>, HitGroupData)
        SHADER_PARAMETER_STRUCT_REF(FLumenHardwareRayTracingUniformBufferParameters, LumenHardwareRayTracingUniformBuffer)
    END_SHADER_PARAMETER_STRUCT()

    class FUseThreadGroupSize64 : SHADER_PERMUTATION_BOOL("RAY_TRACING_USE_THREAD_GROUP_SIZE_64");
    class FUseTracingFeedback   : SHADER_PERMUTATION_BOOL("ENABLE_TRACING_FEEDBACK");
    class FNaniteRayTracing     : SHADER_PERMUTATION_BOOL("NANITE_RAY_TRACING");
    using FBasePermutationDomain = TShaderPermutationDomain<
        FUseThreadGroupSize64, FUseTracingFeedback, FNaniteRayTracing>;

    static bool UseThreadGroupSize64(EShaderPlatform ShaderPlatform);
};
```

### FSharedParameters メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `SceneTextures` | `FSceneTextureParameters` | GBuffer テクスチャ（Depth, Normal, etc.）|
| `SceneTexturesStruct` | `FSceneTextureUniformParameters` UB | SceneTextures の Uniform Buffer 版 |
| `Substrate` | `FSubstrateGlobalUniformParameters` UB | Substrate（旧 Strata）マテリアルシステム用 |
| `TLAS` | `RaytracingAccelerationStructure` | メインシーンの TLAS（Top-Level AS）|
| `FarFieldTLAS` | `RaytracingAccelerationStructure` | Far Field 専用の TLAS |
| `RayTracingSceneMetadata` | `StructuredBuffer` | RT シーンのプリミティブメタデータ |
| `RWInstanceHitCountBuffer` | `RWStructuredBuffer<uint>` | ヒットカウントバッファ（デバッグ/フィードバック用）UAV |
| `NaniteRayTracing` | `FNaniteRayTracingUniformParameters` UB | Nanite RT パラメータ（プロキシメッシュ情報）|
| `LightGridParameters` | `FRayTracingLightGrid` UB | RT 用ライトグリッド（Hit Lighting 時の直接光評価に使用）|
| `ReflectionCapture` | `FReflectionCaptureShaderData` | リフレクションキャプチャデータ |
| `ForwardLightStruct` | `FForwardLightUniformParameters` UB | フォワードライティング情報（Hit Lighting 時）|
| `TracingParameters` | `FLumenCardTracingParameters` | Surface Cache アトラス・フィードバックバッファ一式 |
| `MaxTraversalIterations` | `uint32` | BVH トラバーサルの最大反復数 |
| `MeshSectionVisibilityTest` | `uint32` | メッシュセクションの可視性テスト有効フラグ |
| `MinTraceDistanceToSampleSurfaceCache` | `float` | Surface Cache サンプリングを開始する最小距離 |
| `SurfaceCacheSamplingDepthBias` | `float` | Surface Cache サンプリングの深度バイアス |
| `HitGroupData` | `StructuredBuffer<FHitGroupRootConstants>` | Inline RT 用ヒットグループデータ（プリミティブ固有情報）|
| `LumenHardwareRayTracingUniformBuffer` | UB | Lumen RT 共通 Uniform Buffer |

### FBasePermutationDomain

| パーミュテーション | マクロ定義 | 説明 |
|------------------|-----------|------|
| `FUseThreadGroupSize64` | `RAY_TRACING_USE_THREAD_GROUP_SIZE_64` | PS5 等でスレッドグループ 64 を使うか（デフォルト 32）|
| `FUseTracingFeedback` | `ENABLE_TRACING_FEEDBACK` | Surface Cache フィードバックバッファ書き込みを有効にするか |
| `FNaniteRayTracing` | `NANITE_RAY_TRACING` | Nanite RT（プロキシメッシュ代替）を使うか |

### 使用箇所

- [[ref_lumen_scene_direct_lighting_hwrt]] — `FLumenDirectLightingHardwareRayTracingCS/RGS` が継承
- [[ref_lumen_reflections]] — `FLumenReflectionHardwareRayTracingCS/RGS` が継承
- [[ref_lumen_screen_probe_hwrt]] — `FScreenProbeHardwareRayTracingCS/RGS` が継承
- [[ref_lumen_radiosity]] — Radiosity RT シェーダーが継承

---

## SetLumenHardwareRayTracingSharedParameters

```cpp
void SetLumenHardwareRayTracingSharedParameters(
    FRDGBuilder& GraphBuilder,
    const FSceneTextureParameters& SceneTextures,
    const FViewInfo& View,
    const FLumenCardTracingParameters& TracingParameters,
    FLumenHardwareRayTracingShaderBase::FSharedParameters* SharedParameters);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `SceneTextures` | `const FSceneTextureParameters&` | GBuffer テクスチャ参照 |
| `View` | `const FViewInfo&` | 現在のビュー |
| `TracingParameters` | `const FLumenCardTracingParameters&` | Surface Cache パラメータ（GetLumenCardTracingParameters で取得済み）|
| `SharedParameters` | `FSharedParameters*` | 出力：FSharedParameters 構造体へのポインタ |

### 内部処理フロー

1. **TLAS のバインド**
   ```cpp
   SharedParameters->TLAS = View.RayTracingScene.GetLayerView(ERayTracingSceneLayer::Base);
   SharedParameters->FarFieldTLAS = View.RayTracingScene.GetLayerView(ERayTracingSceneLayer::FarField);
   SharedParameters->RayTracingSceneMetadata = View.RayTracingScene.GetMetadataBuffer(GraphBuilder);
   ```

2. **Nanite RT バインド**
   ```cpp
   SharedParameters->NaniteRayTracing = Nanite::GetRayTracingUniformBuffer(GraphBuilder, View);
   ```

3. **ライティングパラメータ**
   ```cpp
   SharedParameters->LightGridParameters =
       CreateRayTracingLightGrid(GraphBuilder, View, LumenHardwareRayTracing::UseHitLightingDirectLighting());
   SharedParameters->ReflectionCapture = View.ReflectionCaptureUniformBuffer;
   ```

4. **Surface Cache パラメータのコピー**
   ```cpp
   SharedParameters->TracingParameters = TracingParameters;
   SharedParameters->MinTraceDistanceToSampleSurfaceCache =
       GLumenMinTraceDistanceToSampleSurfaceCache;
   ```

5. **Inline RT 用データ**
   ```cpp
   SharedParameters->HitGroupData = View.RayTracingScene.GetHitGroupDataSRV(GraphBuilder);
   SharedParameters->LumenHardwareRayTracingUniformBuffer =
       CreateLumenHardwareRayTracingUniformBuffer(GraphBuilder, View);
   ```

### 使用箇所

- [[ref_lumen_scene_direct_lighting_hwrt]] — Direct Lighting RT シェーダーパラメータ構築時
- [[ref_lumen_reflections]] — Reflection RT シェーダーパラメータ構築時

---

## Lumen::FHitGroupRootConstants

HW RT のヒットグループで使うルート定数。  
Inline RT 時に HitGroupData バッファとして渡される。

```cpp
namespace Lumen {
    struct FHitGroupRootConstants {
        uint32 UserData; // プリミティブ固有データへのインデックス
    };

    enum class ERayTracingShaderDispatchType {
        RayGen = 0, // RayGen シェーダー（フル RT パイプライン）
        Inline = 1  // Inline RT（Compute Shader 内でのトレース）
    };
}
```

### 使用箇所

- [[ref_lumen_hwrt_materials]] — SBT 登録時に各プリミティブの `UserData` をセット
- [[ref_lumen_hwrt_common]] — `FSharedParameters::HitGroupData` バッファの要素型

---

## 主要マクロ

### DECLARE_LUMEN_RAYTRACING_SHADER

`FLumenHardwareRayTracingShaderBase` を継承するシェーダークラスを宣言するマクロ。  
CS（Compute / Inline RT）と RGS（Ray Gen）の2バリアントを自動生成する。

```cpp
#define DECLARE_LUMEN_RAYTRACING_SHADER(ShaderClass) \
    public: \
    using TComputeShaderType = class ShaderClass##CS; \
    using TRayGenShaderType  = class ShaderClass##RGS;
```

### IMPLEMENT_LUMEN_RAYGEN_AND_COMPUTE_RAYTRACING_SHADERS

CS と RGS の両バリアントをまとめて実装するマクロ。  
`AddLumenRayTracingDispatch()` / `AddLumenRayTracingDispatchIndirect()` ヘルパーを含む。

```cpp
// 例: Screen Probe HW RT シェーダーの実装
IMPLEMENT_LUMEN_RAYGEN_AND_COMPUTE_RAYTRACING_SHADERS(FScreenProbeHardwareRayTracingRGS)
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.HardwareRayTracing` | 1 | Lumen 全体の HW RT 有効/無効（0=SW RT にフォールバック）|
| `r.Lumen.HardwareRayTracing.LightingMode` | 0 | ライティングモード（0=Surface Cache, 1=Hit Lighting GI+Refl, 2=Hit Lighting Reflのみ）|
| `r.Lumen.HardwareRayTracing.HitLighting.DirectLighting` | 1 | Hit Lighting でヒット点の Direct Lighting を計算するか |
| `r.Lumen.HardwareRayTracing.HitLighting.ShadowMode` | 2 | シャドウモード（0=無効, 1=ハード, 2=ソフト）|
| `r.Lumen.HardwareRayTracing.HitLighting.Skylight` | 2 | スカイライト（0=無効, 1=有効, 2=Reflection のみ）|
| `r.Lumen.HardwareRayTracing.HitLighting.ReflectionCaptures` | 0 | Hit Lighting でリフレクションキャプチャを使うか |
| `r.Lumen.HardwareRayTracing.HitLighting.ForceOpaque` | false | 全マテリアルをオペーク強制 |

---

## Surface Cache モード vs Hit Lighting モードの比較

| 比較項目 | Surface Cache（LightingMode=0）| Hit Lighting（LightingMode=1/2）|
|---------|------|------|
| GI 品質 | Surface Cache の精度に依存 | ヒット点でフル評価（高品質）|
| GPU コスト | 低い | 非常に高い |
| セカンダリバウンス | Surface Cache を再利用 | Surface Cache を使用 |
| 用途 | 通常ゲーム用途 | 映像制作・高品質モード |
