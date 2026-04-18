# GPU a: TranslucentForward — 半透明フォワードシェーディング

- シェーダー: `BasePassPixelShaders.usf`（Translucent パーミュテーション）, `ComposeSeparateTranslucency.usf`
- CPU 対応: [[a_translucent_rendering]]
- 上位: [[01_translucency_gpu_overview]]

---

## 概要

半透明オブジェクトの描画はフォワードシェーディングパスで行われる。  
Deferred（GBuffer 書き込み）ではなく、ライティング計算をシェーダー内で直接実行する。  
結果は `SeparateTranslucency` テクスチャに書き込まれ、後段で SceneColor に合成される。

---

## BasePassPS（Translucent パーミュテーション）

`BasePassPixelShaders.usf` の Translucent パーミュテーションが使用される。  
不透明の BasePass と同じシェーダーファイルだが、  
`MATERIALBLENDINGMODE_TRANSLUCENT` 等のマクロで半透明専用のコードパスが有効になる。

```hlsl
// 半透明パーミュテーションで有効になるコードパス（概念的）
void BasePassPS(...)
{
    // 1. マテリアル評価（BaseColor, Opacity, Roughness, etc.）
    FMaterialPixelParameters MaterialParameters = GetMaterialPixelParameters(...);

    // 2. フォワードライティング計算
    //    ライティングボリューム（TranslucencyLightingVolume）をサンプリング
    float3 DirectionalLighting = GetTranslucencyLighting(MaterialParameters);

    // 3. SkyLight / Reflection 適用（bApplyLighting == true の場合）
    // 4. Fog / Atmosphere 適用

    // 5. アルファブレンドで出力
    //    - Translucent: アルファブレンド（RGB = 輝度, A = 不透明度）
    //    - Additive: RGB を加算
    //    - Modulate: RGB を乗算
    OutColor = float4(Luminance, Opacity);
}
```

---

## SeparateTranslucency

半透明描画は `SeparateTranslucency` テクスチャに分離して書き込まれることがある。

```
SeparateTranslucency の仕組み:
  ├─ 通常解像度: 最終品質モード
  └─ 半解像度（r.SeparateTranslucencyScreenPercentage = 50）: パフォーマンスモード
       → TSR/TAA のアップスケール前に合成

書き込まれるパス:
  - r.SeparateTranslucency = 1 の場合: 半透明を別テクスチャに書き込む
  - r.SeparateTranslucency = 0 の場合: SceneColor に直接書き込み
```

---

## ComposeSeparateTranslucency（ComposeSeparateTranslucency.usf）

SeparateTranslucency テクスチャを SceneColor に合成する後処理パス。

```hlsl
void MainPS(...)
{
    // 解像度が異なる場合はアップサンプリング
    #if PERMUTATION_NEARESTDEPTHNEIGHBOR
    // 深度ベースの近傍アップサンプリング（SeparateTranslucency.ush）
    float4 SeparateTranslucency = NearestDepthNeighborUpsample(
        SeparateTranslucencyPointTexture,
        SeparateTranslucencyBilinearTexture,
        LowResDepthTexture, FullResDepthTexture, UV);
    #else
    float4 SeparateTranslucency = SeparateTranslucencyBilinearTexture.Sample(...);
    #endif

    // SceneColor に合成（アルファブレンド）
    OutColor = lerp(SceneColorTexture, SeparateTranslucency.rgb, SeparateTranslucency.a);
}
```

---

## CopyBackgroundVisibilityPS

```hlsl
void CopyBackgroundVisibilityPS(...)
```

半透明の背景可視性情報をコピーするパス。  
ホールドアウト（Alpha Holdout）機能と連携して透過領域を管理する。

---

## 深度ソート

半透明オブジェクトは描画前に **CPU 側で深度ソート** される（FTranslucentPrimSet）。  
GPUでの深度ソートは OIT（c グループ）が担当する。

```
FTranslucentPrimSet::DrawPrimitivesParallel()  [CPU: RenderThread]
  │ 視点からの距離でソート済みリスト
  │
  └─ 前から後ろへ順番に描画（Painter's Algorithm）
```

---

## 半透明のライティングモード

| TRANSLUCENCY_LIGHTING_MODE | 内容 |
|---------------------------|------|
| `TLM_VolumetricNonDirectional` | ライティングボリュームをサンプリング（方向性なし）|
| `TLM_VolumetricDirectional` | ライティングボリューム + 法線考慮 |
| `TLM_VolumetricPerVertexNonDirectional` | 頂点ライティング（低コスト）|
| `TLM_Surface` | デファードライティングに近い品質（高コスト）|
| `TLM_SurfacePerPixelLighting` | ピクセル単位のサーフェスライティング |

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.SeparateTranslucency` | 分離半透明バッファの有効/無効 |
| `r.SeparateTranslucencyScreenPercentage` | 分離バッファの解像度スケール（50 = 半解像度）|
| `r.TranslucentSortPolicy` | ソートポリシー（距離・投影距離等）|
| `r.TranslucencyLightingVolumeDim` | ライティングボリュームの解像度 |
