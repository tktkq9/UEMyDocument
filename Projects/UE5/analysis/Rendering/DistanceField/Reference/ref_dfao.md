# Ref: DFAO エントリポイントリファレンス

- 対象: `DistanceFieldAmbientOcclusion.h/.cpp`
- 上位: [[10_distance_field_overview]] | [[c_dfao]]

---

## エントリポイント

### ShouldRenderDistanceFieldLighting

`DistanceFieldAmbientOcclusion.cpp:849`

```cpp
bool ShouldRenderDistanceFieldLighting(
    const FDistanceFieldSceneData& SceneData,
    TConstArrayView<FViewInfo> Views);
```

DFAO を実行可能かどうかを判定する関数。  
`GDistanceFieldAOMultiView`・プラットフォーム機能・オブジェクト数の3条件を確認する。

---

### ShouldRenderDistanceFieldAO

`DistanceFieldAmbientOcclusion.cpp:985`

```cpp
bool ShouldRenderDistanceFieldAO(
    TConstArrayView<FViewInfo> Views,
    const FEngineShowFlags& EngineShowFlags);
```

`r.DistanceFieldAO`・ShowFlags・Lumen 状態・Sky Light 存在を確認する。  
Lumen が有効な場合は `false` を返す。

---

### RenderDFAOAsIndirectShadowing

`DistanceFieldAmbientOcclusion.cpp:872`

```cpp
void FDeferredShadingSceneRenderer::RenderDFAOAsIndirectShadowing(
    FRDGBuilder& GraphBuilder,
    const FSceneTextures& SceneTextures,
    TArray<FRDGTextureRef>& DynamicBentNormalAOTextures);
```

`GDistanceFieldAOApplyToStaticIndirect` && `ShouldRenderDistanceFieldAO()` の場合に  
`RenderDistanceFieldLighting()` を呼び出す。  
Sky Light の `OcclusionMaxDistance` を `FDistanceFieldAOParameters` に設定して渡す。

---

### RenderDistanceFieldLighting

`DistanceFieldAmbientOcclusion.cpp:885`

```cpp
void FSceneRenderer::RenderDistanceFieldLighting(
    FRDGBuilder& GraphBuilder,
    const FSceneTextures& SceneTextures,
    const FDistanceFieldAOParameters& Parameters,
    TArray<FRDGTextureRef>& OutDynamicBentNormalAOTextures,
    bool bVisualizeAmbientOcclusion,
    bool bModulateToScreenSpaceAO);
```

DFAO コーントレースの主要エントリポイント。  
`ShouldRenderDistanceFieldLighting()` が事前 check される。

---

## FDistanceFieldAOParameters

`DistanceFieldAmbientOcclusion.h:33`

```cpp
class FDistanceFieldAOParameters
{
    float GlobalMaxOcclusionDistance; // Global SDF トレース最大距離
    float ObjectMaxOcclusionDistance; // Object SDF 最大距離（現在未使用）
    float Contrast;                    // AO コントラスト係数

    FDistanceFieldAOParameters(float InOcclusionMaxDistance, float InContrast = 0);
};
```

---

## FAOParameters（シェーダーバインド）

`DistanceFieldAmbientOcclusion.h:83`

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FAOParameters, )
    SHADER_PARAMETER(float, AOObjectMaxDistance)
    SHADER_PARAMETER(float, AOStepScale)
    SHADER_PARAMETER(float, AOStepExponentScale)
    SHADER_PARAMETER(float, AOMaxViewDistance)
    SHADER_PARAMETER(float, AOGlobalMaxOcclusionDistance)
END_SHADER_PARAMETER_STRUCT()
```

`DistanceField::SetupAOShaderParameters(AOParameters)` で `FDistanceFieldAOParameters` → `FAOParameters` に変換。

---

## FDFAOUpsampleParameters（アップスケール用）

`DistanceFieldAmbientOcclusion.h:91`

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FDFAOUpsampleParameters, )
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, BentNormalAOTexture)
    SHADER_PARAMETER_SAMPLER(SamplerState, BentNormalAOSampler)
    SHADER_PARAMETER(FVector2f, AOBufferBilinearUVMax)
    SHADER_PARAMETER(float, DistanceFadeScale)
    SHADER_PARAMETER(float, AOMaxViewDistance)
END_SHADER_PARAMETER_STRUCT()
```

半解像度の AO テクスチャをアップスケールする際のパラメータ。

---

## FAOScreenGridParameters（コーン可視性バッファ）

`DistanceFieldAmbientOcclusion.h:67`

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FAOScreenGridParameters, )
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWBuffer<uint>, RWScreenGridConeVisibility)
    SHADER_PARAMETER_RDG_BUFFER_SRV(Buffer<uint>, ScreenGridConeVisibility)
    SHADER_PARAMETER(FIntPoint, ScreenGridConeVisibilitySize)
END_SHADER_PARAMETER_STRUCT()
```

---

## FTileIntersectionParameters（タイル交差リスト）

`DistanceFieldAmbientOcclusion.h:43`

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FTileIntersectionParameters, )
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<FVector4f>, RWTileConeAxisAndCos)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<FVector4f>, RWTileConeDepthRanges)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>, RWNumCulledTilesArray)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>, RWCulledTilesStartOffsetArray)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWBuffer<uint>, RWCulledTileDataArray)
    ...
    SHADER_PARAMETER(FIntPoint, TileListGroupSize)
END_SHADER_PARAMETER_STRUCT()
```

---

## 補助関数

| 関数 | ファイル | 説明 |
|------|---------|------|
| `CullObjectsToView()` | `DistanceFieldAmbientOcclusion.h:110` | ビューに対してオブジェクトをカリング |
| `BuildTileObjectLists()` | `DistanceFieldAmbientOcclusion.h:111` | タイル交差オブジェクトリスト構築 |
| `GetSpacedVectors()` | `DistanceFieldAmbientOcclusion.h:73` | フレームごとに回転するコーン方向セット |
| `GetBufferSizeForAO()` | `DistanceFieldAmbientOcclusion.h:31` | AO バッファ解像度（`GAODownsampleFactor=2` で半解像度）|
| `GetMaxAOViewDistance()` | `DistanceFieldAmbientOcclusion.h:75` | 最大ビュー距離（fp16 上限: 65000.0f）|

---

## 定数

| 定数 | 値 | 説明 |
|------|----|------|
| `GAODownsampleFactor` | `2` | AO 計算の解像度ダウンスケール係数 |
| `CulledTileDataStride` | `2` | タイルデータのストライド |
| `ConeTraceObjectsThreadGroupSize` | `64` | コーントレース CS スレッドグループサイズ |

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.DistanceFieldAO` | DFAO 有効/無効 |
| `r.AOQuality` | 品質レベル（コーン数・ステップ数） |
| `r.AOGlobalDFStartDistance` | Global SDF トレース開始距離 |
| `r.AOMaxViewDistance` | 最大ビュー距離 |
| `r.DistanceFieldAO.ApplyToStaticIndirect` | 静的間接ライティングへ適用 |
| `r.GDistanceFieldAOMultiView` | マルチビュー（VR）対応 |
