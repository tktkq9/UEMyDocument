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

### 使用箇所

- [[ref_lumen_visualize]] — `AddVisualizeLumenScenePass()` 内でモード値に応じて描画パスを切り替え
- [[ref_lumen_core]] — `Lumen::ShouldVisualizeScene(ShowFlags)` がこれらの定数と組み合わせて使用

---

## LumenVisualize 名前空間

### FTonemappingParameters

> **概要**: 可視化パスでのトーンマッピング設定。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FTonemappingParameters, )
    SHADER_PARAMETER(int32, Tonemap)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, EyeAdaptationBuffer)
    SHADER_PARAMETER_RDG_TEXTURE(Texture3D, ColorGradingLUT)
    SHADER_PARAMETER_SAMPLER(SamplerState, ColorGradingLUTSampler)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Tonemap` | `int32` | トーンマップを適用するか（1=適用、0=リニアのまま表示）|
| `EyeAdaptationBuffer` | `StructuredBuffer<float4>` | 自動露出バッファ（HDR 輝度の調整）|
| `ColorGradingLUT` | `Texture3D` | カラーグレーディング LUT テクスチャ |
| `ColorGradingLUTSampler` | `SamplerState` | LUT 用サンプラー |

### 使用箇所

- [[ref_lumen_visualize]] — `FSceneParameters` に内包され、可視化シェーダー全体で共有

---

### FSceneParameters

> **概要**: 可視化シェーダーへの全入力パラメータを集約した構造体。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FSceneParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FTonemappingParameters, TonemappingParameters)
    SHADER_PARAMETER_STRUCT_INCLUDE(ShaderPrint::FShaderParameters, ShaderPrintUniformBuffer)
    SHADER_PARAMETER(FIntPoint, InputViewSize)
    SHADER_PARAMETER(FIntPoint, InputViewOffset)
    SHADER_PARAMETER(FIntPoint, OutputViewSize)
    SHADER_PARAMETER(FIntPoint, OutputViewOffset)
    SHADER_PARAMETER(int32, VisualizeHiResSurface)
    SHADER_PARAMETER(int32, VisualizeMode)
    SHADER_PARAMETER(uint32, VisualizeCullingMode)
    SHADER_PARAMETER_STRUCT_INCLUDE(LumenReflections::FCompositeParameters, ReflectionsCompositeParameters)
    SHADER_PARAMETER_TEXTURE(Texture2D, PreIntegratedGF)
    SHADER_PARAMETER_SAMPLER(SamplerState, PreIntegratedGFSampler)
    SHADER_PARAMETER(uint32, MaxReflectionBounces)
    SHADER_PARAMETER(uint32, MaxRefractionBounces)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, MeshCardsIndexToCardSharingIdBuffer)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `TonemappingParameters` | `FTonemappingParameters` | トーンマッピング設定 |
| `ShaderPrintUniformBuffer` | `ShaderPrint::FShaderParameters` | GPU デバッグテキスト出力用パラメータ |
| `InputViewSize` | `FIntPoint` | 入力ビューの解像度 |
| `InputViewOffset` | `FIntPoint` | 入力ビューのオフセット |
| `OutputViewSize` | `FIntPoint` | 出力ビューの解像度 |
| `OutputViewOffset` | `FIntPoint` | 出力ビューのオフセット |
| `VisualizeHiResSurface` | `int32` | 高解像度 Surface Cache を表示するか |
| `VisualizeMode` | `int32` | 可視化モード番号（`VISUALIZE_MODE_*` 定数）|
| `VisualizeCullingMode` | `uint32` | カリングモードの可視化設定 |
| `ReflectionsCompositeParameters` | `LumenReflections::FCompositeParameters` | 反射合成パラメータ（MaxRoughnessToTrace 等）|
| `PreIntegratedGF` | `Texture2D` | GGX 積分テーブル（反射ビュー表示用）|
| `PreIntegratedGFSampler` | `SamplerState` | GGX 積分テーブル用サンプラー |
| `MaxReflectionBounces` | `uint32` | 最大反射バウンス数 |
| `MaxRefractionBounces` | `uint32` | 最大屈折バウンス数 |
| `MeshCardsIndexToCardSharingIdBuffer` | `Buffer<uint>` | Mesh Card インデックス → Card 共有 ID マッピング |

### 使用箇所

- [[ref_lumen_visualize]] — `AddVisualizeLumenScenePass()` の内部でシェーダーパラメータとして設定

---

### レイアウト定数

```cpp
constexpr int32 NumOverviewTilesPerRow = 3;   // Overview モードの横タイル数
constexpr int32 OverviewTileMargin = 4;        // タイル間マージン（px）
```

| 定数名 | 値 | 説明 |
|-------|-----|------|
| `NumOverviewTilesPerRow` | 3 | `VISUALIZE_MODE_OVERVIEW` での横タイル数（Surface Cache / 法線 / アルベド等を並べる）|
| `OverviewTileMargin` | 4 | タイル間の余白（ピクセル）|

---

### LumenVisualize 名前空間の関数

| 関数シグネチャ | 役割 |
|-------------|------|
| `VisualizeHardwareRayTracing(GraphBuilder, Scene, SceneTextures, View, FrameTemporaries, TracingParameters, IndirectTracingParameters, VisualizeParameters, SceneColor, bVisualizeModeWithHitLighting, DiffuseIndirectMethod)` | HW RT デバッグ可視化パスを RDG に追加 |
| `IsHitLightingForceEnabled(View, DiffuseIndirectMethod)` | Hit Lighting 強制有効か判定 |
| `UseHitLighting(View, DiffuseIndirectMethod)` | Hit Lighting を使うか判定 |
| `UseSurfaceCacheFeedback(ShowFlags)` | Surface Cache Feedback を使うか判定 |

### 使用箇所

- [[ref_lumen_visualize]] — `AddVisualizeLumenScenePass()` から `VisualizeHardwareRayTracing()` が呼ばれる
- [[ref_lumen_reflection_hwrt]] — `LumenVisualize::UseHitLighting()` と同じ判定ロジックを共有（`LumenReflections::UseHitLighting()` もある）
- [[ref_lumen_surface_cache_feedback]] — `UseSurfaceCacheFeedback()` でフィードバックパスの有効化を制御

---

## AddVisualizeLumenScenePass

```cpp
FScreenPassTexture AddVisualizeLumenScenePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    EDiffuseIndirectMethod DiffuseIndirectMethod,
    EReflectionsMethod ReflectionsMethod,
    const FVisualizeLumenSceneInputs& Inputs,
    FLumenSceneFrameTemporaries& FrameTemporaries);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `View` | `const FViewInfo&` | カメラビュー（`VisualizeMode` を取得するために使用）|
| `DiffuseIndirectMethod` | `EDiffuseIndirectMethod` | Diffuse GI 方式（Lumen / SSGI 等）|
| `ReflectionsMethod` | `EReflectionsMethod` | 反射方式（Lumen / SSR 等）|
| `Inputs` | `const FVisualizeLumenSceneInputs&` | SceneColor / Depth / LUT 等の入力テクスチャ |
| `FrameTemporaries` | `FLumenSceneFrameTemporaries&` | フレームスコープの一時リソース群 |

### 戻り値

`FScreenPassTexture` — 可視化結果が書き込まれたスクリーンパステクスチャ

### 使用箇所

- `FDeferredShadingSceneRenderer` — フレーム末尾にデバッグビジュアライゼーションパスとして呼ばれる

### 内部処理フロー

1. **モード判定**
   ```cpp
   int32 VisualizeMode = GetLumenVisualizeMode(View);
   // VisualizeMode == 0 なら何もせずリターン
   ```

2. **共通パラメータ構築**
   ```cpp
   LumenVisualize::FSceneParameters SceneParameters;
   SceneParameters.VisualizeMode = VisualizeMode;
   // Tonemapping / View矩形 / 反射パラメータを設定
   ```

3. **モード別パス追加**
   ```cpp
   if (VisualizeMode == VISUALIZE_MODE_LUMEN_SCENE) {
       // Card 配置可視化シェーダーを Dispatch
   } else if (VisualizeMode == VISUALIZE_MODE_SURFACE_CACHE) {
       // Surface Cache アトラス表示
   } else if (VisualizeMode == VISUALIZE_MODE_REFLECTION_VIEW) {
       // 反射レイ可視化（HW RT モード時は VisualizeHardwareRayTracing も呼ぶ）
       LumenVisualize::VisualizeHardwareRayTracing(GraphBuilder, ...);
   }
   ```

4. **Overview タイル合成**
   ```cpp
   if (VisualizeMode == VISUALIZE_MODE_OVERVIEW) {
       // NumOverviewTilesPerRow × N グリッドに複数バッファを並べて表示
   }
   ```

---

## GetLumenVisualizeMode

> [!note]- GetLumenVisualizeMode — 現在の可視化モード番号を返す
>
> ```cpp
> int32 GetLumenVisualizeMode(const FViewInfo& View);
> ```
>
> `r.Lumen.Visualize.Mode` の値を返す。`View` のショーフラグが可視化を許可している場合のみ非ゼロを返す。
>
> **使用箇所**: [[ref_lumen_visualize]] — `AddVisualizeLumenScenePass()` の冒頭で呼ばれ、0 なら早期リターン

---

## FVisualizeLumenSceneInputs

> **概要**: `AddVisualizeLumenScenePass()` への入力テクスチャ・バッファ群。

```cpp
struct FVisualizeLumenSceneInputs {
    FScreenPassRenderTarget OverrideOutput;
    FScreenPassTexture SceneColor;
    FScreenPassTexture SceneDepth;
    FRDGTextureRef ColorGradingTexture;
    FRDGBufferRef  EyeAdaptationBuffer;
    FSceneTextureShaderParameters SceneTextures;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `OverrideOutput` | `FScreenPassRenderTarget` | 出力先のオーバーライド（省略時は内部で新規作成）|
| `SceneColor` | `FScreenPassTexture` | シーンカラー入力（可視化モードによっては上書き）|
| `SceneDepth` | `FScreenPassTexture` | シーン深度入力 |
| `ColorGradingTexture` | `FRDGTextureRef` | カラーグレーディング LUT テクスチャ |
| `EyeAdaptationBuffer` | `FRDGBufferRef` | 自動露出バッファ |
| `SceneTextures` | `FSceneTextureShaderParameters` | GBuffer 等のシーンテクスチャ群 |

### 使用箇所

- [[ref_lumen_visualize]] — `AddVisualizeLumenScenePass()` の `Inputs` 引数として渡される
- `FDeferredShadingSceneRenderer` — SceneColor / Depth / LUT を設定して呼び出す

---

## 利用方法（エディタでの可視化）

コンソールコマンドで切り替え：

```
r.Lumen.Visualize.Mode 5   → Surface Cache アトラスを表示
r.Lumen.Visualize.Mode 3   → Lumen Scene の Card 配置を表示
r.Lumen.Visualize.Mode 8   → アルベドアトラスを表示
r.Lumen.Visualize.Mode 1   → 全体概要（デフォルト）
```
