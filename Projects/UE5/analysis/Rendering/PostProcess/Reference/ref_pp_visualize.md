# REF: Visualize 系ファイル群

- 対象ファイル（11本）:
  - `PostProcessVisualizeBuffer.h/.cpp`
  - `PostProcessVisualizeHDR.h/.cpp`
  - `PostProcessVisualizeComplexity.h/.cpp`
  - `PostProcessVisualizeShadingModels.cpp`（← `VisualizeShadingModels.cpp`）
  - `PostProcessVisualizeLocalExposure.h/.cpp`
  - `PostProcessVisualizeNanite.h/.cpp`
  - `PostProcessVisualizeVirtualTexture.h/.cpp`
  - `PostProcessVisualizeCalibrationMaterial.h/.cpp`
  - `PostProcessVisualizeLevelInstance.h/.cpp`
  - `VisualizeMotionVectors.h/.cpp`
  - `VisualizeTemporalUpscaler.cpp`
- 関連Details: [[g_pp_misc]]

---

## 概要

GBuffer・シェーディング情報・パフォーマンス統計等を**デバッグ表示する Visualize パス群**。  
各パスは `Show Flags`（`r.ShowFlag.*`）または `r.VisualizeXxx` CVar で有効化する。

---

## PostProcessVisualizeBuffer.h/.cpp

**GBuffer の各チャンネルを画面に可視化する**。`vis` コンソールコマンドで対象を切り替える。

```cpp
// GBuffer チャンネル可視化パス
FScreenPassTexture AddVisualizeGBufferOverviewPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FVisualizeGBufferOverviewInputs& Inputs);

// 単一バッファ可視化（任意チャンネル選択）
FScreenPassTexture AddVisualizeBufferPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    const FRDGTextureRef BufferToVisualize);
```

| 可視化対象の例 | 内容 |
|-------------|------|
| `BaseColor` | GBuffer A のアルベド |
| `WorldNormal` | GBuffer B の法線 |
| `Roughness` | GBuffer B のラフネス |
| `Metallic` | GBuffer B のメタリック |
| `ShadingModel` | GBuffer B のシェーディングモデル ID |
| `Depth` | シーン深度 |
| `Velocity` | ベロシティ |

---

## PostProcessVisualizeHDR.h/.cpp

**HDR 輝度ヒストグラムと EyeAdaptation の状態**を可視化する。

```cpp
FScreenPassTexture AddVisualizeHDRPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FVisualizeHDRInputs& Inputs);

struct FVisualizeHDRInputs
{
    FScreenPassTexture SceneColor;
    FRDGBufferRef EyeAdaptationBuffer;
    FRDGBufferRef HistogramBuffer;
    float AverageSceneLuminance;
};
```

---

## PostProcessVisualizeComplexity.h/.cpp

**シェーダー命令数（描画負荷）を色分けして可視化**する。

```cpp
FScreenPassTexture AddShaderComplexityPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    FScreenPassTexture ComplexityInput);

// 色のグラデーション: 青（低負荷）→ 緑 → 赤 → 白（過負荷）
```

---

## VisualizeShadingModels.cpp

**各ピクセルのシェーディングモデルを色分け表示**する（GBuffer B の ShadingModelID を参照）。

```cpp
FScreenPassTexture AddVisualizeShadingModelPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    const FMinimalSceneTextures& SceneTextures);
```

---

## PostProcessVisualizeLocalExposure.h/.cpp

**ローカル露出マップを可視化**する（明るい部分→赤, 暗い部分→青）。

```cpp
FScreenPassTexture AddVisualizeLocalExposurePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    FRDGTextureRef BlurredLogLuminanceTexture,
    FRDGBufferRef EyeAdaptationBuffer);
```

---

## PostProcessVisualizeNanite.h/.cpp

**Nanite の各バッファ（VisBuffer / MaterialDepth / ClusterDepth 等）を可視化**する。

```cpp
FScreenPassTexture AddVisualizeNanitePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    const Nanite::FRasterResults& NaniteRasterResults);
```

---

## PostProcessVisualizeVirtualTexture.h/.cpp

**Virtual Texture のストリーミング状態を可視化**する（ページキャッシュ状況等）。

```cpp
FScreenPassTexture AddVisualizeVirtualTexturePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input);
```

---

## PostProcessVisualizeCalibrationMaterial.h/.cpp

**カラーキャリブレーション用マテリアル**の可視化パス。TV キャリブレーション画面等に使用。

```cpp
FScreenPassTexture AddVisualizeCalibrationMaterialPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FVisualizeCalibrationMaterialInputs& Inputs);
```

---

## PostProcessVisualizeLevelInstance.h/.cpp

**レベルインスタンスの境界を可視化**する（エディタ機能）。

```cpp
FScreenPassTexture AddVisualizeLevelInstancePass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    const FMinimalSceneTextures& SceneTextures);
```

---

## VisualizeMotionVectors.h/.cpp

**ベロシティバッファを可視化**する（速度方向→色相, 速度大きさ→明度）。

```cpp
FScreenPassTexture AddVisualizeMotionVectorsPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    FRDGTextureRef SceneVelocity);
```

---

## VisualizeTemporalUpscaler.cpp

**TSR / TAA の品質情報（ゴースト強度・鮮明度スコア等）を可視化**する。

```cpp
FScreenPassTexture AddVisualizeTemporalUpscalerPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor);
```

---

## 主要 CVar まとめ

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.ShowFlag.VisualizeBuffer` | 0 | バッファ可視化 |
| `r.VisualizeOcclusionQueries` | 0 | オクルージョンクエリ可視化 |
| `r.VisualizeHDR` | 0 | HDR ヒストグラム可視化 |
| `r.Nanite.Visualize` | "" | Nanite 可視化モード |
| `r.ShowFlag.ShaderComplexity` | 0 | シェーダー複雑度表示 |
| `r.ShowFlag.ShadingModels` | 0 | シェーディングモデル表示 |
| `r.VisualizeLocalExposure` | 0 | ローカル露出可視化 |
| `r.VisualizeTemporalUpscaler` | 0 | TSR 品質可視化 |
