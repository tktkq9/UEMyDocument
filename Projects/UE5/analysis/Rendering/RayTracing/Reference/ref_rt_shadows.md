# リファレンス：RayTracingShadows.h / RayTracingAmbientOcclusion.cpp / RaytracingSkylight.cpp

- グループ: b - シャドウ・AO・スカイライト
- 上位: [[b_rt_shadow_ao]]
- 関連: [[ref_rt_scene]] | [[ref_rt_sbt]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/RayTracing/RayTracingShadows.h/.cpp`
  - `Engine/Source/Runtime/Renderer/Private/RayTracing/RayTracingAmbientOcclusion.cpp`
  - `Engine/Source/Runtime/Renderer/Private/RayTracing/RaytracingSkylight.cpp`

## 概要

HW レイトレーシングを使ったシャドウ・AO・スカイライトの3パス実装。  
それぞれ独立したレイジェネレーションシェーダー（RGS）とデノイザーを持つ。

---

## RT シャドウ（RayTracingShadows）

### `RayTracingShadows::SetRayTracingSceneOptions`

```cpp
namespace RayTracingShadows
{
    void SetRayTracingSceneOptions(
        bool bSceneHasLightsWithRayTracedShadows,
        RayTracing::FSceneOptions& SceneOptions);
}
```

| 引数 | 型 | 説明 |
|------|-----|------|
| `bSceneHasLightsWithRayTracedShadows` | `bool` | RT シャドウを持つ光源がシーンにあるか |
| `SceneOptions` | `RayTracing::FSceneOptions&` | インスタンス収集オプション（in/out） |

RT シャドウが必要な場合に `FSceneOptions` の関連フラグを立てて、  
TLAS のインスタンス収集段階でシャドウキャスターを含めるよう制御する。

### 主要 CVar（RayTracingShadows）

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RayTracing.Shadows` | -1 | -1=自動, 0=無効, 1=有効 |
| `r.RayTracing.Shadows.EnableTwoSidedGeometry` | 1 | シャドウレイで両面判定 |
| `r.RayTracing.Shadows.EnableMaterials` | 1 | AHS（Any-Hit Shader）有効化 |

---

## RT AO（RayTracingAmbientOcclusion）

### `ShouldRenderRayTracingAmbientOcclusion`

```cpp
bool ShouldRenderRayTracingAmbientOcclusion(const FViewInfo& View);
```

| ロジック | 説明 |
|---------|------|
| `GRayTracingAmbientOcclusion < 0` | PostProcess Volume の `RayTracingAO` 値を使用 |
| `GRayTracingAmbientOcclusion != 0` | CVar の値を直接使用 |
| `RayTracingAOIntensity > 0` | 強度が 0 より大きい場合のみ有効 |

最終的に `ShouldRenderRayTracingEffect(bEnabled, FullPipeline, View)` を通して判定。

### `FRayTracingAmbientOcclusionRGS`

```cpp
class FRayTracingAmbientOcclusionRGS : public FGlobalShader
{
    DECLARE_GLOBAL_SHADER(FRayTracingAmbientOcclusionRGS)
    // レイジェネレーションシェーダー
    // AO レイを発射し、ヒット距離からオクルージョン値を計算
};
```

### 主要 CVar（RT AO）

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RayTracing.AmbientOcclusion` | -1 | -1=PostProcess 依存 |
| `r.RayTracing.AmbientOcclusion.SamplesPerPixel` | -1 | SPP（-1=PostProcess 依存） |
| `r.RayTracing.AmbientOcclusion.EnableTwoSidedGeometry` | 0 | 両面判定 |
| `r.RayTracing.AmbientOcclusion.EnableMaterials` | 0 | AHS 有効化 |
| `r.AmbientOcclusion.Denoiser` | 2 | 0=無効, 1=デフォルト, 2=GScreenSpaceDenoiser |

---

## RT スカイライト（RaytracingSkylight）

### 主要関数

| 関数 | 説明 |
|------|------|
| `RenderRayTracingSkyLight()` | スカイライトのシャドウイングパス本体 |
| `ShouldRenderRayTracingSkyLight()` | `r.RayTracing.SkyLight` と光源の有無で判定 |

天球方向へレイを発射し、スカイライトの遮蔽マップを生成。  
テンポラル蓄積（`FViewState::SkyLightHistory`）で前フレームの結果を再利用する。

### 主要 CVar（RT スカイライト）

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RayTracing.SkyLight` | 1 | RT スカイライト有効 |
| `r.RayTracing.SkyLight.SamplesPerPixel` | -1 | SPP |
| `r.RayTracing.SkyLight.EnableTwoSidedGeometry` | 1 | 両面判定 |
| `r.RayTracing.SkyLight.EnableMaterials` | 1 | AHS 有効化 |

---

## 使用箇所

- [[ref_rt_scene]] — `RayTracingShadows::SetRayTracingSceneOptions()` でインスタンス収集フラグを設定
- `FDeferredShadingSceneRenderer::Render()` — 各パスを個別に呼び出す
