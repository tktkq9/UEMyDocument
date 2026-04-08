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
        SurfaceCache,            // Surface Cache からサンプリング（デフォルト）
        HitLighting,             // ヒット点でフルマテリアル評価（GI+Reflection）
        HitLightingForReflections, // ヒット点でフルマテリアル評価（Reflectionのみ）
        MAX
    };
}
```

### ユーティリティ関数

```cpp
namespace LumenHardwareRayTracing {
    bool IsInlineSupported();    // Inline RT（CS内）が使えるプラットフォームか
    bool IsRayGenSupported();    // RayGen シェーダーが使えるか
    float GetFarFieldBias();     // Far Field トレースのバイアス値

    bool UseSurfaceCacheAlphaMasking(); // Surface Cache のアルファマスキングを使うか
    EAvoidSelfIntersectionsMode GetAvoidSelfIntersectionsMode();

    // Hit Lighting
    EHitLightingMode GetHitLightingMode(const FViewInfo& View, EDiffuseIndirectMethod DiffuseIndirectMethod);
    uint32 GetHitLightingShadowMode();           // シャドウモード（0=無効, 1=ハード, 2=ソフト）
    uint32 GetHitLightingShadowTranslucencyMode();
    bool UseHitLightingForceOpaque();            // 全マテリアルをオペーク扱いにするか
    bool UseHitLightingDirectLighting();         // ヒット点で Direct Lighting を計算するか
    bool UseHitLightingSkylight(EDiffuseIndirectMethod);  // スカイライトをヒット点で計算するか
    bool UseReflectionCapturesForHitLighting(); // リフレクションキャプチャを Hit Lighting に使うか
    bool UseShaderExecutionReordering();         // SER (Shader Execution Reordering) を使うか

    // シーンオプションのセットアップ
    void SetRayTracingSceneOptions(const FViewInfo& View,
        EDiffuseIndirectMethod DiffuseIndirectMethod,
        EReflectionsMethod ReflectionsMethod,
        RayTracing::FSceneOptions& SceneOptions);
}
```

---

## FLumenHardwareRayTracingShaderBase

すべての Lumen RT シェーダーの**基底クラス**。`FGlobalShader` を継承。  
TLAS・Nanite RT・ライトグリッド・Surface Cache パラメータをまとめて管理する。

```cpp
class FLumenHardwareRayTracingShaderBase : public FGlobalShader {
public:
    // 全 Lumen RT シェーダーが共有する FSharedParameters
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

    // 基底パーミュテーション（全 Lumen RT シェーダーが共通で持つ）
    class FUseThreadGroupSize64 : SHADER_PERMUTATION_BOOL("RAY_TRACING_USE_THREAD_GROUP_SIZE_64");
    class FUseTracingFeedback   : SHADER_PERMUTATION_BOOL("ENABLE_TRACING_FEEDBACK");
    class FNaniteRayTracing     : SHADER_PERMUTATION_BOOL("NANITE_RAY_TRACING");
    using FBasePermutationDomain = TShaderPermutationDomain<
        FUseThreadGroupSize64, FUseTracingFeedback, FNaniteRayTracing>;

    // スレッドグループサイズ（PS5 等では 64 が有利）
    static bool UseThreadGroupSize64(EShaderPlatform ShaderPlatform);
};
```

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

### IMPLEMENT_LUMEN_COMPUTE_RAYTRACING_SHADER

Inline RT（Compute Shader）バリアントを実装するマクロ。  
`AddLumenRayTracingDispatch()` / `AddLumenRayTracingDispatchIndirect()` ヘルパーを生成する。

```cpp
#define IMPLEMENT_LUMEN_COMPUTE_RAYTRACING_SHADER(ShaderClass) \
    class ShaderClass##CS : public ShaderClass { ... };
```

---

## 主要 CVar（LumenHardwareRayTracingCommon.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.HardwareRayTracing` | 1 | Lumen 全体の HW RT 有効/無効（0 で SW RT にフォールバック）|
| `r.Lumen.HardwareRayTracing.LightingMode` | 0 | ライティングモード（0=Surface Cache, 1=Hit Lighting GI+Refl, 2=Hit Lighting Reflのみ）|
| `r.Lumen.HardwareRayTracing.HitLighting.DirectLighting` | 1 | Hit Lighting で Direct Lighting をヒット点で計算するか |
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
