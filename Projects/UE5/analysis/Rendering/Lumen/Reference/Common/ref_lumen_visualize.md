# リファレンス：LumenVisualize.h/cpp / LumenVisualizeHardwareRayTracing.cpp / LumenVisualizeRadianceCache.cpp

- グループ: Common
- 上位: [[02_lumen_overview]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenVisualize.h/cpp`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenVisualizeHardwareRayTracing.cpp`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenVisualizeRadianceCache.cpp`

---

## 概要

Lumen の**デバッグ可視化**システム。エディタのビューポートで  
`r.Lumen.Visualize.Mode` を変更することで各内部バッファを視覚的に確認できる。

---

## 可視化モード定数（`#define`）

| 定数名 | 値 | 表示内容 |
|-------|-----|---------|
| `VISUALIZE_MODE_OVERVIEW` | 1 | 全体概要（タイル分割表示）|
| `VISUALIZE_MODE_PERFORMANCE_OVERVIEW` | 2 | パフォーマンス計測の概要 |
| `VISUALIZE_MODE_LUMEN_SCENE` | 3 | Lumen Scene（Card の配置）|
| `VISUALIZE_MODE_REFLECTION_VIEW` | 4 | 反射ビュー |
| `VISUALIZE_MODE_SURFACE_CACHE` | 5 | Surface Cache アトラス |
| `VISUALIZE_MODE_GEOMETRY_NORMALS` | 6 | ジオメトリ法線 |
| `VISUALIZE_MODE_DEDICATED_REFLECTION_RAYS` | 7 | 反射レイの専用表示 |
| `VISUALIZE_MODE_ALBEDO` | 8 | アルベドアトラス |
| `VISUALIZE_MODE_NORMALS` | 9 | 法線アトラス |
| `VISUALIZE_MODE_OPACITY` | 11 | 不透明度アトラス |
| `VISUALIZE_MODE_CARD_SHARING_ID` | 22 | Card 共有 ID |
| `VISUALIZE_MODE_SCREENPROBEGATHER_FAST_UPDATE_MODE_AMOUNT` | 23 | 高速更新量 |
| `VISUALIZE_MODE_SCREENPROBEGATHER_NUM_FRAMES_ACCUMULATED` | 24 | 蓄積フレーム数 |

---

## LumenVisualize 名前空間

### シェーダーパラメータ構造体

#### FTonemappingParameters

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FTonemappingParameters, )
    SHADER_PARAMETER(int32, Tonemap)                          // トーンマップを適用するか
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, EyeAdaptationBuffer) // 自動露出バッファ
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, ColorGradingLUT) // カラーグレーディング LUT
    SHADER_PARAMETER_SAMPLER(SamplerState, ColorGradingLUTSampler)
END_SHADER_PARAMETER_STRUCT()
```

#### FSceneParameters

可視化シェーダーへの入力パラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FSceneParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FTonemappingParameters, TonemappingParameters)
    SHADER_PARAMETER_STRUCT_INCLUDE(ShaderPrint::FShaderParameters, ShaderPrintUniformBuffer)
    SHADER_PARAMETER(FIntPoint, InputViewSize)       // 入力ビューサイズ
    SHADER_PARAMETER(FIntPoint, InputViewOffset)     // 入力ビューオフセット
    SHADER_PARAMETER(FIntPoint, OutputViewSize)      // 出力ビューサイズ
    SHADER_PARAMETER(FIntPoint, OutputViewOffset)    // 出力ビューオフセット
    SHADER_PARAMETER(int32, VisualizeHiResSurface)   // 高解像度 Surface Cache を表示するか
    SHADER_PARAMETER(int32, VisualizeMode)           // 可視化モード番号
    SHADER_PARAMETER(uint32, VisualizeCullingMode)   // カリングモードの可視化
    SHADER_PARAMETER_STRUCT_INCLUDE(LumenReflections::FCompositeParameters, ReflectionsCompositeParameters)
    SHADER_PARAMETER_TEXTURE(Texture2D, PreIntegratedGF)
    SHADER_PARAMETER_SAMPLER(SamplerState, PreIntegratedGFSampler)
    SHADER_PARAMETER(uint32, MaxReflectionBounces)   // 最大反射バウンス数
    SHADER_PARAMETER(uint32, MaxRefractionBounces)   // 最大屈折バウンス数
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, MeshCardsIndexToCardSharingIdBuffer)
END_SHADER_PARAMETER_STRUCT()
```

### レイアウト定数

```cpp
constexpr int32 NumOverviewTilesPerRow = 3;   // Overview モードの横タイル数
constexpr int32 OverviewTileMargin = 4;        // タイル間マージン（px）
```

### 関数

| 関数シグネチャ | 役割 |
|-------------|------|
| `VisualizeHardwareRayTracing(GraphBuilder, Scene, SceneTextures, View, FrameTemporaries, TracingParameters, IndirectTracingParameters, VisualizeParameters, SceneColor, bVisualizeModeWithHitLighting, DiffuseIndirectMethod)` | HW RT デバッグ可視化パスを追加 |
| `IsHitLightingForceEnabled(View, DiffuseIndirectMethod)` | Hit Lighting 強制有効か |
| `UseHitLighting(View, DiffuseIndirectMethod)` | Hit Lighting を使うか |
| `UseSurfaceCacheFeedback(ShowFlags)` | Surface Cache Feedback を使うか |

---

## グローバル関数（LumenVisualize.h）

| 関数 | 役割 |
|------|------|
| `AddVisualizeLumenScenePass(GraphBuilder, View, DiffuseIndirectMethod, ReflectionsMethod, Inputs, FrameTemporaries)` | Lumen シーン全体の可視化パスを RDG に追加して結果テクスチャを返す |
| `GetLumenVisualizeMode(View)` | 現在の可視化モード番号を返す |

---

## FVisualizeLumenSceneInputs（入力構造体）

```cpp
struct FVisualizeLumenSceneInputs {
    FScreenPassRenderTarget OverrideOutput; // [Optional] 出力先（省略時は新規作成）
    FScreenPassTexture SceneColor;          // [Required] シーンカラー
    FScreenPassTexture SceneDepth;          // [Required] シーン深度
    FRDGTextureRef ColorGradingTexture;     // カラーグレーディングテクスチャ
    FRDGBufferRef  EyeAdaptationBuffer;     // 自動露出バッファ
    FSceneTextureShaderParameters SceneTextures; // GBuffer 等のシーンテクスチャ
};
```

---

## 利用方法（エディタでの可視化）

コンソールコマンドで切り替え：

```
r.Lumen.Visualize.Mode 5   → Surface Cache アトラスを表示
r.Lumen.Visualize.Mode 3   → Lumen Scene の Card 配置を表示
r.Lumen.Visualize.Mode 8   → アルベドアトラスを表示
r.Lumen.Visualize.Mode 1   → 全体概要（デフォルト）
```
