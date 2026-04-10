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

> **概要**: 反射トレース方式の設定（不透明 / フロントレイヤーで共用）

```cpp
struct FLumenReflectionsConfig {
    EReflectionsMethod ReflectionsMethod;
    bool bUseLumenReflections;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `ReflectionsMethod` | `EReflectionsMethod` | 反射トレース方式（Lumen SW RT / HW RT / SSR 等）|
| `bUseLumenReflections` | `bool` | Lumen 反射を使用するか（`false` = SSR のみ）|

### 使用箇所

- [[ref_lumen_front_layer]] — `RenderLumenFrontLayerTranslucencyReflections()` の引数として渡され、SW / HW RT の分岐に使用
- [[ref_lumen_reflections]] — 不透明反射パスでも同じ構造体でトレース方式を管理

---

## FLumenFrontLayerTranslucency

> **概要**: フロントレイヤー透明反射の出力テクスチャ群。`RenderLumenFrontLayerTranslucencyReflections()` の戻り値。

```cpp
struct FLumenFrontLayerTranslucency {
    FRDGTexture* Radiance;
    FRDGTexture* Mask;
    FRDGTexture* Weight;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Radiance` | `FRDGTexture*` | 透明前面の反射ラジアンス（float3 / RGBA16F）|
| `Mask` | `FRDGTexture*` | 反射が有効なピクセルのフラグ（タイル処理・合成パスで利用）|
| `Weight` | `FRDGTexture*` | 合成ブレンドウェイト（ラフネスベースのフェード値）|

### 使用箇所

- [[ref_lumen_front_layer]] — `RenderLumenFrontLayerTranslucencyReflections()` で生成・返却
- [[ref_lumen_front_layer]] — `CompositeLumenFrontLayerTranslucencyReflections()` で `Radiance * Weight` として SceneColor に合成

---

## FFrontLayerTranslucencyData

> **概要**: フロントレイヤー反射の入力データ。透明前面の GBuffer 相当データを保持する。

```cpp
struct FFrontLayerTranslucencyData {
    FRDGTexture* FrontLayerDepth;
    FRDGTexture* FrontLayerNormal;
    FRDGTexture* FrontLayerRoughness;
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `FrontLayerDepth` | `FRDGTexture*` | 透明オブジェクト最前面の深度テクスチャ（`RenderFrontLayerTranslucency()` で生成）|
| `FrontLayerNormal` | `FRDGTexture*` | 最前面のワールドスペース法線テクスチャ |
| `FrontLayerRoughness` | `FRDGTexture*` | 最前面のラフネステクスチャ（反射するか・フェードするかの判定に使用）|

### 使用箇所

- [[ref_lumen_front_layer]] — `RenderLumenFrontLayerTranslucencyReflections()` の `FrontLayerData` 引数として渡される
- 深度テクスチャは `RenderFrontLayerTranslucency()` が不透明描画パス後に生成

---

## RenderLumenFrontLayerTranslucencyReflections

```cpp
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
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（SDF / TLAS アクセス）|
| `View` | `const FViewInfo&` | カメラビュー |
| `SceneTextures` | `const FSceneTextures&` | HZB スクリーントレース用テクスチャ |
| `FrontLayerData` | `const FFrontLayerTranslucencyData&` | 透明前面の Depth / Normal / Roughness |
| `TracingParameters` | `const FLumenCardTracingParameters&` | Surface Cache アトラス・トレース共通パラメータ |
| `RadianceCacheParameters` | `FRadianceCacheInterpolationParameters` | ラフネス大きいピクセルのフォールバック照明 |
| `ReflectionsConfig` | `const FLumenReflectionsConfig&` | SW / HW RT の選択設定 |
| `ComputePassFlags` | `ERDGPassFlags` | AsyncCompute / Compute の選択フラグ |

### 戻り値

`FLumenFrontLayerTranslucency` — Radiance / Mask / Weight の3テクスチャ群

### 使用箇所

- [[ref_lumen_reflections]] — `RenderLumenReflections()` の前段で呼ばれ、透明前面の反射を計算
- [[ref_lumen_front_layer]] — 戻り値は `CompositeLumenFrontLayerTranslucencyReflections()` に渡す

### 内部処理フロー

1. **深度・法線・ラフネスの取得**
   ```cpp
   // FFrontLayerTranslucencyData から透明前面の GBuffer を参照
   FrontLayerData.FrontLayerDepth / FrontLayerNormal / FrontLayerRoughness
   ```

2. **反射レイ生成**
   ```cpp
   // フロントレイヤーの法線 + ラフネスから GGX 重み付きレイを生成
   // UseJitter = true でブルーノイズジッター適用
   GenerateReflectionRaysCS(...)
   ```

3. **トレース方式の選択**
   ```cpp
   if (ReflectionsConfig.ReflectionsMethod == HW_RT) {
       RenderLumenHardwareRayTracingReflections(...);
   } else {
       TraceReflections(...);  // SW RT（Mesh SDF / Global SDF）
   }
   ```

4. **ラジアンス解決**
   ```cpp
   // ヒット点の FinalLightingAtlas をサンプリング
   // FrontLayerRoughness > MaxRoughnessToTrace の場合: Radiance Cache で補間
   if (bUseRadianceCache) {
       ApplyRadianceCacheToReflections(...);
   }
   ```

5. **結果テクスチャの生成**
   ```cpp
   // Radiance / Mask / Weight を出力してリターン
   return FLumenFrontLayerTranslucency { Radiance, Mask, Weight };
   ```

---

## CompositeLumenFrontLayerTranslucencyReflections

> [!note]- CompositeLumenFrontLayerTranslucencyReflections — 反射を SceneColor に合成
>
> ```cpp
> void CompositeLumenFrontLayerTranslucencyReflections(
>     FRDGBuilder& GraphBuilder,
>     const FViewInfo& View,
>     const FSceneTextures& SceneTextures,
>     const FLumenFrontLayerTranslucency& FrontLayerTranslucencyReflections);
> ```
>
> **処理**: `FrontLayerTranslucency.Radiance * Weight` を SceneColor に加算合成する。  
> `Mask` を使って反射が有効なピクセルにのみ適用。
>
> **パラメータ**
>
> | 引数 | 型 | 説明 |
> |------|-----|------|
> | `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
> | `View` | `const FViewInfo&` | カメラビュー |
> | `SceneTextures` | `const FSceneTextures&` | 書き込み先 SceneColor を含むテクスチャ群 |
> | `FrontLayerTranslucencyReflections` | `const FLumenFrontLayerTranslucency&` | Radiance / Mask / Weight の入力 |
>
> **使用箇所**: [[ref_lumen_reflections]] — `RenderLumenReflections()` の後段で `SceneColor` への合成に呼ばれる

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
