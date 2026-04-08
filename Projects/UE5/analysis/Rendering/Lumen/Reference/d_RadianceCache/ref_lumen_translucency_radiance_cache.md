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
    // 透明マテリアルへの Radiance Cache Reflections を使うか
    // 条件: r.Lumen.TranslucencyReflections.RadianceCache != 0
    //        && EngineShowFlags.LumenReflections
    bool UseLumenTranslucencyRadianceCacheReflections(const FSceneViewFamily& ViewFamily);

    // 透明マテリアルをマークパスに含めるべきか（マテリアル単位）
    // 条件: IsTranslucentBlendMode()
    //        && (TLM_Surface || TLM_SurfacePerPixelLighting || VoxelMarking)
    //        && ShouldRenderInMainPass()
    bool ShouldRenderInTranslucencyRadianceCacheMarkPass(
        bool bShouldRenderInMainPass, const FMaterial& Material);
}
```

---

## FLumenTranslucencyRadianceCacheMarkPassUniformParameters

透明マテリアルのマークパスで使うグローバルユニフォームバッファ。

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FLumenTranslucencyRadianceCacheMarkPassUniformParameters, )
    SHADER_PARAMETER_STRUCT(FSceneTextureUniformParameters, SceneTextures)
    SHADER_PARAMETER_STRUCT_INCLUDE(FHZBParameters, HZBParameters)
    SHADER_PARAMETER(float, HZBMipLevel)
    SHADER_PARAMETER(uint32, UseHZBTest)    // HZB オクルージョンテストを使うか

    // マーク対象のプローブを記録する RWTexture3D パラメータ
    SHADER_PARAMETER_STRUCT_INCLUDE(LumenRadianceCache::FRadianceCacheMarkParameters, RadianceCacheMarkParameters)
    SHADER_PARAMETER(uint32, MarkRadianceCache)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

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
  ├─ [Mark Pass] 透明マテリアルを描画してプローブをマーク
  │   → MarkDownsampleFactor でダウンサンプルした解像度で描画
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
