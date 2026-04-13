# a: MegaLights パイプライン全体

- 対象ファイル: `MegaLights.h/.cpp` / `MegaLightsInternal.h`
- 概要: [[06_megalights_overview]]

---

## 概要

`RenderMegaLights()` は **確率的サンプリング方式で多数のダイナミックライトを処理するパス**。  
RenderLights() の直後に呼ばれ、MegaLights 対象として分類されたライトの  
照明計算を GBuffer / HairStrands の各入力ソースに対して実行する。

---

## 呼び出し位置

```cpp
// DeferredShadingRenderer.cpp:3314
RenderLights(GraphBuilder, SceneTextures, LightingChannelsTexture, SortedLightSet);

// DeferredShadingRenderer.cpp:3318
if (MegaLights::IsEnabled(ViewFamily))
{
    RenderMegaLights(
        GraphBuilder,
        MegaLightsFrameTemporaries,
        SceneTextures,
        NaniteShadingMasks,
        LightingChannelsTexture);
}
```

---

## RenderMegaLights() フロー

```
RenderMegaLights()                              MegaLights.cpp:1959
  │
  └─ for each ViewIndex:
      ├─ ViewContext     = GBuffer 入力（EMegaLightsInput::GBuffer）
      ├─ ViewContextHair = HairStrands 入力（EMegaLightsInput::HairStrands）
      │
      ├─ RenderMegaLightsViewContext(ViewContext, ...)
      │   → GBuffer メッシュへの MegaLights 適用 + Volume 生成
      │
      └─ [HairStrands有効]
          RenderMegaLightsViewContext(ViewContextHair, ...)
```

---

## RenderMegaLightsViewContext() の内部パス順

```
RenderMegaLightsViewContext()                   MegaLights.cpp:1920
  │
  ├─ [A] DownsampleSceneDepth / WorldNormal
  │   → 半解像度（2x2 ダウンサンプル）のDepth/Normal テクスチャを生成
  │
  ├─ [B] GenerateLightSamples                   MegaLightsSampling.cpp
  │   → FGenerateLightSamplesCS
  │   → 各ピクセルに対してランダムにライトをサンプリング
  │   → LightSamples テクスチャ（サンプルされたライスト + 方向）
  │
  ├─ [C] VisibilityTest（シャドウ）
  │   ├─ [EnabledRT]  MegaLights::RayTraceLightSamples()
  │   │               HW RT でシャドウレイを発射
  │   └─ [EnabledVSM] MegaLights::MarkVSMPages() → VSM 経由
  │
  ├─ [D] Resolve                                MegaLightsResolve.cpp
  │   → FMegaLightsResolveCS
  │   → LightSamples を BRDF 評価して輝度に変換
  │   → 全サンプルを合算・正規化
  │
  ├─ [E] Denoise                                MegaLightsDenoising.cpp
  │   ├─ Temporal: テンポラル蓄積（History ブレンド）
  │   └─ Spatial:  空間フィルタ
  │
  └─ [F] [Volume有効] TranslucencyVolume        MegaLights.cpp
      → 半透明オブジェクト用の 3D ボリュームに照明を注入
```

---

## EMegaLightsInput（入力ソース）

```cpp
// MegaLightsInternal.h:82
enum class EMegaLightsInput : uint8
{
    GBuffer,      // 通常のオブジェクト（GBuffer から Normal 等を取得）
    HairStrands,  // HairStrands 専用パス（VisibilityData.SampleLightingTexture に書き込む）
    Count
};
```

---

## ダウンサンプル戦略

```cpp
// r.MegaLights.DownsampleMode
// 0: Disabled (1x1) → フルレス
// 1: Checkerboard (2x1) → 水平方向のみ半解像度
// 2: Half-resolution (2x2) → 縦横ともに半解像度（デフォルト）

FIntPoint DownsampleFactor = MegaLights::GetDownsampleFactorXY(MaterialSource, ShaderPlatform);
// GBuffer 通常: (2,2)
// HairStrands: マテリアルソース依存
```

---

## 関連リファレンス

- [[ref_megalights_core]] — `FMegaLightsFrameTemporaries` / `FMegaLightsVolume`
- [[b_megalights_sampling]] — サンプリング詳細
- [[c_megalights_shadow]] — シャドウ統合詳細
- [[d_megalights_resolve]] — Resolve / Denoise 詳細

---

## TileClassify → Stochastic → Shadow → Resolve 詳細フロー

```
【TileClassify フェーズ】
  FMegaLightsTileClassifyCS（MegaLights.cpp）
    │
    per 8x8 タイル:
    ├─ GBuffer.ShadingModelID を読み取り
    ├─ タイル内最大 Roughness を計算
    ├─ ETileType を決定:
    │   ShadingModelID が Simple 系 → SimpleShading
    │   それ以外                   → ComplexShading
    │   Rect Light が影響範囲内    → *_Rect / *_Rect_Textured
    │   Substrate 有効             → SingleShading / ComplexSpecialShading
    └─ TileClassificationMark テクスチャ（R8, タイルあたり1テクセル）に書き込み

【Stochastic Sampling フェーズ】
  FGenerateLightSamplesCS（MegaLightsSampling.cpp）
    │
    per pixel（ダウンサンプル済み解像度）:
    ├─ ClusteredLightGrid[ClusterKey] → Froxel 内のライスト一覧
    ├─ 各ライストの Irradiance（強度 × 投影面積）を推定
    │   → 重要度（PDF）比例サンプリングリスト構築
    │   → r.MegaLights.GuideByHistory > 0 の場合:
    │       前フレームの VisibleLightHash でライストの可視性を考慮してPDF補正
    ├─ BlueNoise + FrameIndex でランダムインデックスを生成
    ├─ PDF に従って NumSamplesPerPixel 個のライストをサンプリング
    └─ LightSamples（インデックス + PDF weight）/ LightSampleRays（方向・TMax）に書き込み

【Shadow フェーズ】
  → 詳細は [[c_megalights_shadow]] 参照
  [EnabledRT]
    MegaLights::RayTraceLightSamples()
      → TLAS へシャドウレイ → Visibility bit を LightSamples に書き込む
  [EnabledVSM]
    MegaLights::MarkVSMPages()（GenerateSamples より前に呼ばれる）
      → VirtualShadowMapArray に必要ページをマーク
      → RenderVirtualShadowMaps() でページを実際にレンダリング
    Resolve フェーズで VSM テクスチャから Shadow 値を読み込み

【Resolve フェーズ】
  FMegaLightsResolveCS（MegaLightsResolve.cpp）
    │
    per pixel（ダウンサンプル解像度）:
    ├─ LightSamples から (LightIndex, Weight, Visibility) を取得
    ├─ ForwardLightData[LightIndex] → ライスト色・強度・位置
    ├─ GBuffer（フルレゾ or ダウンサンプル）→ BaseColor / Normal / Roughness
    ├─ BRDF評価:
    │   Diffuse  = Lambertian(BaseColor) × NdotL
    │   Specular = GGX(F0, Roughness, Normal, ViewDir, LightDir)
    │   Irradiance = (Diffuse + Specular) × LightColor × Visibility / PDF
    ├─ NumSamplesPerPixel 個の Irradiance を平均化
    └─ ResolvedRadiance テクスチャ（R11G11B10F）に書き込み
```
