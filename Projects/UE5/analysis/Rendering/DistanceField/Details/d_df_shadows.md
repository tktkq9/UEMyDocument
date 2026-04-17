# D: Distance Field Shadows — Mesh SDF コーントレースによるソフトシャドウ

- 対象: `DistanceFieldShadowing.cpp`, `DistanceFieldLightingShared.h`, `ScenePrivate.h`
- 上位: [[10_distance_field_overview]]
- Reference: [[ref_df_shadows]]

---

## 概要

**Distance Field Shadows（DF Shadows）** は Mesh SDF を使ったコーントレースにより  
ディレクショナルライト・スポットライトのソフトシャドウを生成する。  
Ray Traced Distance Field Shadows とも呼ばれ、VSM や HW RayTracing と共存可能。

---

## 有効条件

```
ShouldPrepareForDistanceFieldShadows() が true になるための条件:
  ✓ r.DistanceFieldShadowing == 1
  ✓ EngineShowFlags.DynamicShadows == true
  ✓ SupportsDistanceFieldShadows() （プラットフォーム・機能レベル）
  ✓ DoesPlatformSupportDistanceFieldShadowing(ShaderPlatform)
```

`DistanceFieldShadowing.cpp:764`

---

## Mesh SDF コーントレースの仕組み

```
【入力】
  Object SDF バッファ（FDistanceFieldSceneData からアップロード済み）
  Mesh SDF Atlas テクスチャ（各メッシュの SDF を詰め込んだ 3D アトラス）

【コーントレースフロー】
  1. シャドウ投影プリミティブを収集（FProjectedShadowInfo）
  2. シャドウ領域を Scissor でクリップ
  3. Object SDF バッファから影響オブジェクトをカリング
  4. 各ピクセルからライト方向へコーントレース
     → コーン角度 = ライトの光源サイズ（ソフト係数）
  5. トレース距離に応じた遮蔽率を計算
  6. RayTracedShadowsTexture へ書き込み

【出力】
  RayTracedShadowsTexture（ScreenShadowMask）
```

---

## 静的・動的オブジェクト別の戦略

| オブジェクト種別 | SDF 種別 | 更新頻度 |
|--------------|---------|---------|
| 静的スタティックメッシュ | Mesh SDF（Atlas に常駐） | 変化時のみ再アップロード |
| 動的スタティックメッシュ | Mesh SDF（Atlas から毎フレーム更新） | トランスフォーム変更時 |
| スケルタルメッシュ | DF Shadows 非対応（Shadow Map 使用） | — |

---

## FProjectedShadowInfo::RenderRayTracedDistanceFieldProjection

`DistanceFieldShadowing.cpp:910, 1121`

```cpp
// 内部でシャドウテクスチャを生成して返すオーバーロード
FScreenPassTexture FProjectedShadowInfo::RenderRayTracedDistanceFieldProjection(
    FRDGBuilder& GraphBuilder,
    bool bAsyncCompute,
    const FMinimalSceneTextures& SceneTextures,
    const FViewInfo& View,
    FIntRect ScissorRect);

// 結果を ScreenShadowMaskTexture へ書き込むオーバーロード
void FProjectedShadowInfo::RenderRayTracedDistanceFieldProjection(
    FRDGBuilder& GraphBuilder,
    const FMinimalSceneTextures& SceneTextures,
    FRDGTextureRef ScreenShadowMaskTexture,
    const FViewInfo& View,
    FIntRect ScissorRect,
    FTiledShadowRendering* TiledShadowRendering);
```

---

## RayTraceShadows（内部コア処理）

`DistanceFieldShadowing.cpp:797`

```cpp
void RayTraceShadows(
    FRDGBuilder& GraphBuilder,
    bool bAsyncCompute,
    const FMinimalSceneTextures& SceneTextures,
    ...);
```

Object SDF バッファをサンプリングして Compute Shader でコーントレースを実行する中核関数。  
`DistanceFieldShadowing.usf` シェーダーを使用。

---

## VSM / Lumen との共存

| シャドウシステム | 共存 | 備考 |
|--------------|------|------|
| Virtual Shadow Maps（VSM） | ✓ 共存可能 | 個々のライトで独立して切り替え |
| Hardware Ray Tracing Shadow | ✓ 共存可能 | DF Shadows より高品質 |
| Lumen GI | ✓ 共存可能 | Lumen は独自の Shadow 計算も持つ |
| Mesh Distance Field Shadow Off | — | ライト単位で `bUseRaytracedDistanceFieldShadows` で制御 |

ライトコンポーネントの `bUseRaytracedDistanceFieldShadows` を true にすることで  
そのライトに限り DF Shadows を有効化できる。

---

## コード実行フロー

```
FDeferredShadingSceneRenderer::RenderShadowDepthMaps()  [DeferredShadingRenderer.cpp]
  │
  └─ RenderShadowProjections()
       │
       └─ FProjectedShadowInfo::RenderProjection()
            │ ← UsesDistanceFieldShadows() が true の場合
            │
            └─ RenderRayTracedDistanceFieldProjection(GraphBuilder, SceneTextures,
                                                       ScreenShadowMaskTexture, View, ScissorRect)
                 │
                 ├─ CullDistanceFieldObjectsForLight()  // ライト影響オブジェクトのカリング
                 │
                 ├─ RayTraceShadows(GraphBuilder, bAsyncCompute, SceneTextures, ...)
                 │   → DistanceFieldShadowing.usf（Compute Shader）
                 │       ├─ Object SDF バッファをサンプリング
                 │       ├─ Mesh SDF Atlas テクスチャをサンプリング
                 │       └─ コーントレース結果を出力テクスチャへ書き込み
                 │
                 └─ RayTracedShadowsTexture → ScreenShadowMaskTexture へ合成
```

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.DistanceFieldShadowing` | DF Shadows 全体の有効/無効（0: 無効, 1: 有効） |
| `r.DFShadowQuality` | シャドウ品質（ステップ数・コーン数） |
| `r.DistanceFieldShadowing.UseAsyncCompute` | 非同期 Compute 使用フラグ |
| `r.DistanceFieldShadowing.MaxObjectsPerShadow` | 影響オブジェクト数上限 |
