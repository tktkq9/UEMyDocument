# リファレンス：LumenFrontLayerTranslucency.h / LumenFrontLayerTranslucency.cpp

- グループ: f - Reflections
- 上位: [[f_lumen_reflections]]
- 関連: [[ref_lumen_reflections]] | [[ref_ray_traced_translucency]] | [[ref_lumen_radiance_cache]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenFrontLayerTranslucency.h`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenFrontLayerTranslucency.cpp`

---

## 概要

透明マテリアル（ガラス・水面など）の**最前面レイヤーに対して反射を適用**するシステム。  
不透明サーフェスの反射処理（LumenReflections）とは別パスで処理し、  
透明オブジェクトの前面の法線・ラフネスを用いた反射レイトレースを担う。

---

## FLumenReflectionsConfig

```cpp
// 反射設定（不透明 / フロントレイヤーで共用）
struct FLumenReflectionsConfig {
    // 反射トレースに Lumen を使用するか
    EReflectionsMethod ReflectionsMethod;

    // Hit Lighting を使うか（HW RT 専用）
    bool bUseLumenReflections;
};
```

---

## FLumenFrontLayerTranslucency

```cpp
// フロントレイヤー透明反射の出力テクスチャ群
struct FLumenFrontLayerTranslucency {
    // 反射ラジアンス（透明オブジェクト前面の反射色）
    FRDGTexture* Radiance;

    // 反射確認済みフラグ（タイル処理用）
    FRDGTexture* Mask;

    // 反射合成に使う重み（ラフネスベースのフェード）
    FRDGTexture* Weight;
};
```

---

## FFrontLayerTranslucencyData

```cpp
// フロントレイヤー処理に必要な入力データ
struct FFrontLayerTranslucencyData {
    // フロントレイヤーの深度テクスチャ（透明オブジェクトの最前面）
    FRDGTexture* FrontLayerDepth;

    // フロントレイヤーの法線テクスチャ（ワールドスペース法線）
    FRDGTexture* FrontLayerNormal;

    // ラフネス（反射するか / フェードするかの判定に使用）
    FRDGTexture* FrontLayerRoughness;
};
```

---

## エントリポイント

```cpp
// フロントレイヤー透明反射のトレース
FLumenFrontLayerTranslucency RenderLumenFrontLayerTranslucencyReflections(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    const FFrontLayerTranslucencyData& FrontLayerData,
    const FLumenCardTracingParameters& TracingParameters,
    const LumenRadianceCache::FRadianceCacheInterpolationParameters& RadianceCacheParameters,
    const FLumenReflectionsConfig& ReflectionsConfig,
    ERDGPassFlags ComputePassFlags);

// フロントレイヤー反射を最終カラーに合成
void CompositeLumenFrontLayerTranslucencyReflections(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    const FLumenFrontLayerTranslucency& FrontLayerTranslucencyReflections);
```

---

## フロントレイヤー処理フロー

```
RenderLumenFrontLayerTranslucencyReflections()
  │
  ├─ [深度・法線・ラフネスの取得]
  │   FFrontLayerTranslucencyData から透明前面の GBuffer を参照
  │   FrontLayerDepth / FrontLayerNormal / FrontLayerRoughness
  │
  ├─ [反射レイ生成]
  │   フロントレイヤーの法線 + ラフネスから GGX 重み付きレイを生成
  │
  ├─ [トレース方式の選択]
  │   ReflectionsConfig.ReflectionsMethod に従い分岐:
  │   ├─ SW RT → TraceReflections（Mesh SDF / Global SDF）
  │   └─ HW RT → RenderLumenHardwareRayTracingReflections
  │
  ├─ [ラジアンス解決]
  │   ヒット点の FinalLightingAtlas をサンプリング
  │   ラフネスが MaxRoughnessToTrace を超える場合: Radiance Cache で補間
  │
  └─ FLumenFrontLayerTranslucency { Radiance, Mask, Weight } を返す

CompositeLumenFrontLayerTranslucencyReflections()
  │
  └─ Radiance * Weight を SceneColor に加算合成
```

---

## 透明前面深度の生成

```
フロントレイヤー深度テクスチャは別途:
  RenderFrontLayerTranslucency()（LumenFrontLayerTranslucency.cpp）
  が不透明描画パスの後に実行し、透明オブジェクトの最前面のみを Z バッファに書き込む。
  
  これにより、複数の透明レイヤーが重なる場合も最前面の法線・ラフネスのみを
  反射計算に使用できる。
```

---

## 主要 CVar（LumenFrontLayerTranslucency.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.FrontLayer.Translucency.Reflections` | 1 | フロントレイヤー反射の有効/無効 |
| `r.Lumen.FrontLayer.Translucency.Reflections.MaxRoughness` | 0.4 | 反射を適用する最大ラフネス |
| `r.Lumen.FrontLayer.Translucency.Reflections.DownsampleFactor` | 1 | 解像度ダウンサンプル係数 |
| `r.Lumen.FrontLayer.Translucency.ScreenTraces` | 1 | スクリーンスペーストレースの有効/無効 |
