# C: Distance Field AO（DFAO） — GlobalSDF コーントレースによる AO

- 対象: `DistanceFieldAmbientOcclusion.h/.cpp`, `DistanceFieldLightingShared.h`
- 上位: [[10_distance_field_overview]]
- Reference: [[ref_dfao]]

---

## 概要

**DFAO（Distance Field Ambient Occlusion）** は Global SDF を用いたコーントレースにより、  
Sky Light の遮蔽（Ambient Occlusion）を計算するシステム。  
Lumen が有効な場合は DFAO は自動的に無効化される。

---

## 有効条件

```
ShouldRenderDistanceFieldAO() が true になるための条件:
  ✓ r.DistanceFieldAO == 1
  ✓ シーンに Sky Light が存在する
  ✓ DistanceFields.NumObjectsInBuffer > 0（SDF オブジェクトが存在）
  ✗ Lumen が有効な場合は false を返す
  ✗ SCS_SceneDepth 等の UnlitCapture では false

ShouldRenderDistanceFieldLighting() が true になるための条件:
  ✓ GDistanceFieldAOMultiView == true または View 数 == 1
  ✓ SupportDistanceFieldAO（プラットフォーム・機能レベル）
  ✓ SceneData.NumObjectsInBuffer > 0
```

---

## FDistanceFieldAOParameters

`DistanceFieldAmbientOcclusion.h:33`

```cpp
class FDistanceFieldAOParameters
{
public:
    float GlobalMaxOcclusionDistance; // Global SDF コーントレースの最大距離
    float ObjectMaxOcclusionDistance; // Object SDF コーントレースの最大距離（未使用）
    float Contrast;                    // AO コントラスト係数

    FDistanceFieldAOParameters(float InOcclusionMaxDistance, float InContrast = 0);
};
```

`OcclusionMaxDistance` は Sky Light の `OcclusionMaxDistance` プロパティから取得される。

---

## FAOParameters（シェーダーパラメータ）

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FAOParameters, )
    SHADER_PARAMETER(float, AOObjectMaxDistance)      // オブジェクト単位の最大距離
    SHADER_PARAMETER(float, AOStepScale)              // コーンステップのスケール
    SHADER_PARAMETER(float, AOStepExponentScale)      // ステップ指数スケール
    SHADER_PARAMETER(float, AOMaxViewDistance)        // 最大ビュー距離（fp16 限界）
    SHADER_PARAMETER(float, AOGlobalMaxOcclusionDistance) // Global SDF 最大距離
END_SHADER_PARAMETER_STRUCT()
```

---

## コーントレース仕組み

```
【入力】
  Global SDF クリップマップ（GDF_Full）
  GAODownsampleFactor = 2（解像度 1/2 でトレース）

【コーントレースフロー】
  1. タイル単位でオブジェクトリストをカリング（CullObjectsToView）
  2. 各ピクセルから複数コーン方向（空間分散サンプリング）でトレース
     → GetSpacedVectors() で1フレームずつ回転するサンプルパターン
  3. コーン内の SDF を最小値でアキュムレート
  4. 結果を BentNormal と AO として出力

【出力】
  BentNormal テクスチャ（ScreenSpace、半解像度）
  → Upscale → Sky Light に適用
```

---

## タイル交差処理

`DistanceFieldAmbientOcclusion.h:111`

```cpp
extern void BuildTileObjectLists(
    FRDGBuilder& GraphBuilder,
    const FScene& Scene,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    FRDGBufferRef ObjectIndirectArguments,
    const FDistanceFieldCulledObjectBufferParameters& CulledObjectBufferParameters,
    FTileIntersectionParameters TileIntersectionParameters,
    FRDGTextureRef DistanceFieldNormal,
    const FDistanceFieldAOParameters& Parameters);
```

スクリーンを 32x32 ピクセルのタイルに分割し、各タイルに影響する SDF オブジェクトのリストを Compute Shader で構築する。

---

## Temporal フィルタとの統合

```
フレームごとに GetSpacedVectors() で異なるコーン方向セットを生成
  → 隣接フレームのサンプルをアキュムレートして高品質な AO を実現
  → Temporal AA に類似したアプローチ
```

`FAOScreenGridParameters::ScreenGridConeVisibility` に1フレームのコーン可視性を保存。

---

## Lumen との共存関係

```
Lumen 有効時:
  ShouldRenderDistanceFieldAO() → false
  → DFAO パスは完全にスキップ
  → AO は Lumen の GI 計算内で処理される

DFAO 単独使用時:
  Sky Light の間接シャドウとして AO テクスチャを適用
  RenderDFAOAsIndirectShadowing() で BasePass 後に合成
```

---

## コード実行フロー

`DeferredShadingRenderer.cpp` / `DistanceFieldAmbientOcclusion.cpp`

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─ UpdateGlobalDistanceFieldVolume()  ← Global SDF 更新
  │
  └─ RenderDFAOAsIndirectShadowing()   [line 872]
       │ ← GDistanceFieldAOApplyToStaticIndirect &&
       │   ShouldRenderDistanceFieldAO() &&
       │   ShouldRenderDistanceFieldLighting()
       │
       └─ RenderDistanceFieldLighting(GraphBuilder, SceneTextures,
                                      FDistanceFieldAOParameters(...),
                                      DynamicBentNormalAOTextures,
                                      bVisualizeAO=true, bModulateToSS=false)
            │
            ├─ CullObjectsToView()            // オブジェクトをカリング
            ├─ BuildTileObjectLists()          // タイル交差リスト構築
            ├─ RenderDistanceFieldAOScreenGrid()// コーントレース（CS）
            │   → GlobalDistanceField.usf 使用
            └─ UpsampleBentNormalAO()          // 解像度復元
                 → DynamicBentNormalAOTextures に格納
                    → Sky Light 合成に渡される
```

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.DistanceFieldAO` | DFAO 有効/無効（0: 無効, 1: 有効） |
| `r.AOQuality` | AO 品質（コーン数・ステップ数） |
| `r.AOGlobalDFStartDistance` | Global SDF トレース開始距離 |
| `r.AOMaxViewDistance` | DFAO の最大ビュー距離 |
| `r.DistanceFieldAO.ApplyToStaticIndirect` | 静的間接ライティングへの適用 |
