# リファレンス：LumenTranslucencyRadianceCache.cpp

- グループ: d - Radiance Cache
- 上位: [[d_lumen_radiance_cache]]
- 関連: [[ref_lumen_radiance_cache]] | [[ref_lumen_translucency_volume]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenTranslucencyRadianceCache.cpp`

---

## 概要

**透明マテリアル**（Translucent Surface）への Lumen Reflections を提供するファイル。  
Radiance Cache を使い、透明サーフェスが占める画面領域をマークして  
必要なプローブをトレース・補間することで反射照明を計算する。

---

## Lumen 名前空間（Translucency RC 判定）

```cpp
namespace Lumen {
    bool UseLumenTranslucencyRadianceCacheReflections(const FSceneViewFamily& ViewFamily);
    bool ShouldRenderInTranslucencyRadianceCacheMarkPass(
        bool bShouldRenderInMainPass, const FMaterial& Material);
}
```

### UseLumenTranslucencyRadianceCacheReflections

| 引数 | 型 | 説明 |
|------|-----|------|
| `ViewFamily` | `const FSceneViewFamily&` | ビューファミリー（ShowFlags の参照）|

**戻り値**: `bool`

**有効条件**:
- `r.Lumen.TranslucencyReflections.RadianceCache != 0`
- `EngineShowFlags.LumenReflections` が有効

### 使用箇所（UseLumenTranslucencyRadianceCacheReflections）

- [[ref_lumen_reflections]] — `RenderLumenReflections()` 内で透明マテリアルへの反射パスを追加するか判定
- [[ref_lumen_translucency_radiance_cache]] — `RenderLumenTranslucencyRadianceCache()` の有効チェックに使用

---

### ShouldRenderInTranslucencyRadianceCacheMarkPass

透明マテリアルをマークパスに含めるかマテリアル単位で判定する関数。

| 引数 | 型 | 説明 |
|------|-----|------|
| `bShouldRenderInMainPass` | `bool` | メインパスで描画対象になっているか |
| `Material` | `const FMaterial&` | 判定対象のマテリアル |

**戻り値**: `bool`

**有効条件**（すべて満たす場合に true）:
- `bShouldRenderInMainPass == true`
- `IsTranslucentBlendMode(Material)` が true
- ブレンドモードが `TLM_Surface` / `TLM_SurfacePerPixelLighting` のいずれか、または `bIsSky` が false
- `Material.ShouldRenderInMainPass()` が true

### 使用箇所（ShouldRenderInTranslucencyRadianceCacheMarkPass）

- [[ref_lumen_translucency_radiance_cache]] — マークパスのドローコール生成時に対象マテリアルをフィルタリング

---

## FLumenTranslucencyRadianceCacheMarkPassUniformParameters

透明マテリアルのマークパスで使うグローバルユニフォームバッファ。

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FLumenTranslucencyRadianceCacheMarkPassUniformParameters, )
    SHADER_PARAMETER_STRUCT(FSceneTextureUniformParameters, SceneTextures)
    SHADER_PARAMETER_STRUCT_INCLUDE(FHZBParameters, HZBParameters)
    SHADER_PARAMETER(float, HZBMipLevel)
    SHADER_PARAMETER(uint32, UseHZBTest)
    SHADER_PARAMETER_STRUCT_INCLUDE(LumenRadianceCache::FRadianceCacheMarkParameters, RadianceCacheMarkParameters)
    SHADER_PARAMETER(uint32, MarkRadianceCache)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `SceneTextures` | `FSceneTextureUniformParameters` | Depth バッファ等の GBuffer テクスチャ参照 |
| `HZBParameters` | `FHZBParameters` | HZB（Hierarchical Z Buffer）のテクスチャ・サンプラ設定 |
| `HZBMipLevel` | `float` | HZBOcclusionTest で使用するミップレベル（ダウンサンプル量に対応）|
| `UseHZBTest` | `uint32` | HZB オクルージョンテストを行うか（bool として使用）。`r.Lumen.TranslucencyReflections.HZBOcclusionTest` に対応 |
| `RadianceCacheMarkParameters` | `FRadianceCacheMarkParameters` | マーク対象プローブの 3D インディレクションテクスチャ UAV と座標変換パラメータ |
| `MarkRadianceCache` | `uint32` | Radiance Cache のマーク処理を実行するか（bool として使用）|

### 使用箇所

- [[ref_lumen_translucency_radiance_cache]] — `RenderLumenTranslucencyRadianceCacheMarkPass()` でグローバル UB として全透明マテリアルシェーダーにバインド
- [[ref_lumen_radiance_cache]] — `FRadianceCacheMarkParameters` をインクルードしてマーク処理を統合

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.TranslucencyReflections.RadianceCache` | 1 | 透明マテリアルへの Radiance Cache Reflections の有効/無効 |
| `r.Lumen.TranslucencyReflections.MarkDownsampleFactor` | 4 | マークパスのダウンサンプル係数（2の冪; 小さいほど正確だがコスト増）|
| `r.Lumen.TranslucencyReflections.HZBOcclusionTest` | true | マーク時に HZB オクルージョンテストを行うか |
| `r.Lumen.TranslucencyReflections.ReprojectionRadiusScale` | 10 | リプロジェクション半径スケール（大きいほど遠くのキャッシュを参照）|
| `r.Lumen.TranslucencyReflections.ClipmapFadeSize` | 4.0 | クリップマップ間のディザ遷移サイズ（プローブ単位）|

---

## パイプライン全体フロー

```
RenderLumenTranslucencyRadianceCache()
  │
  ├─ [有効チェック]
  │   UseLumenTranslucencyRadianceCacheReflections() が false → 早期リターン
  │
  ├─ [Mark Pass] 透明マテリアルを描画してプローブをマーク
  │   → MarkDownsampleFactor でダウンサンプルした解像度で描画
  │   → ShouldRenderInTranslucencyRadianceCacheMarkPass() でマテリアルをフィルタ
  │   → HZBOcclusionTest が有効なら、深度バッファで不可視領域をスキップ
  │   → FLumenTranslucencyRadianceCacheMarkPassUniformParameters でプローブ座標を記録
  │
  ├─ [UpdateRadianceCaches()]
  │   → マークされたプローブを Radiance Cache として更新
  │   → GI 用 Radiance Cache と並行して GPU ディスパッチをオーバーラップ
  │
  └─ [Reflection Composite]
        → FRadianceCacheInterpolationParameters を Translucent BasePass シェーダーに渡す
        → FLumenTranslucencyLightingParameters.RadianceCacheInterpolationParameters として使用
```

---

## TLM_Surface と TLM_SurfacePerPixelLighting の違い

| モード | 説明 | Radiance Cache |
|-------|------|---------------|
| `TLM_Surface` | 頂点単位でライティング | Radiance Cache から補間（高速）|
| `TLM_SurfacePerPixelLighting` | ピクセル単位でライティング | Radiance Cache から補間（高品質）|
| その他 | Volume GI ベース | Translucency Volume Lighting を使用 |

---

## MarkDownsampleFactor の効果

```
MarkDownsampleFactor = 4（デフォルト）:
  → 1920×1080 の画面を 480×270 でマーク
  → パフォーマンス優先、大きなオブジェクトは確実にカバー

MarkDownsampleFactor = 1:
  → フルスクリーン解像度でマーク（精度最高、コスト最大）
  → 小さな透明オブジェクトも確実にカバー
```

---

## HZB オクルージョンテストの仕組み

```
UseHZBTest = true の場合:
  各透明ピクセルの深度値を HZB と比較
  → 不透明ジオメトリに完全に隠されたピクセルはマークをスキップ
  → Radiance Cache の無駄な更新を削減

HZBMipLevel:
  MarkDownsampleFactor に対応するミップレベルを使用
  → ダウンサンプル済みのマークパスに合わせた解像度で HZB をサンプリング
```
