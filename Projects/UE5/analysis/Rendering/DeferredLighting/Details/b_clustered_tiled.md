# b: Clustered / Tiled Deferred Shading

- 対象ファイル: `ClusteredDeferredShadingRenderer.h/.cpp`
- 概要: [[16_deferred_lighting_overview]]

---

## 概要

多数のローカルライトを効率よく処理するための**バッチ照明計算パス**。  
画面を 3D クラスターグリッドに分割し、各クラスターに対応するライストを  
Compute Shader でまとめて処理する。1ライトあたりの Draw Call コストを排除できる。

| 方式 | 分割単位 | 実装 |
|-----|---------|------|
| **Clustered Deferred** | 3D クラスター（奥行き方向も分割）| `FClusteredDeferredLightingCS` |
| **Tiled Deferred** | 2D タイル（奥行き方向は考慮しない）| 旧方式、現在は Clustered が優先 |

---

## ライトグリッド構築

```
// まず GatherLightsForView() でライトをグリッドに注入する
FForwardLightingViewResources::InjectLightsIntoGrid()
  │
  └─ 各ライトを FForwardLightData に変換
      → FForwardLightUniformParameters::NumLocalLights
      → LightDataBuffer（StructuredBuffer<FForwardLightData>）
      → ClusteredLightGrid（各クラスターのライトインデックス配列）
```

---

## AddClusteredDeferredShadingPass()

```cpp
// ClusteredDeferredShadingRenderer.cpp
void AddClusteredDeferredShadingPass(
    FRDGBuilder& GraphBuilder,
    FMinimalSceneTextures& SceneTextures,
    const FSortedLightSetSceneInfo& SortedLightSet,
    FRDGTextureRef VirtualShadowMapMaskBits,
    FRDGTextureRef VirtualShadowMapMaskBitsHairStrands,
    FRDGTextureRef LightingChannelsTexture)
{
    // FClusteredDeferredLightingCS を Dispatch
    // スレッドグループ: 8x8 ピクセルタイル
    TShaderMapRef<FClusteredDeferredLightingCS> ComputeShader(View.ShaderMap, PermutationVector);
    FComputeShaderUtils::AddPass(GraphBuilder, ..., ComputeShader, Parameters,
        FIntVector(TilesX, TilesY, 1));
}
```

---

## FClusteredDeferredLightingCS の動作

```
FClusteredDeferredLightingCS（Compute Shader）
  per-tile（8x8 ピクセル）:
  │
  ├─ SceneDepth から View Space Z を取得
  ├─ クラスター Z インデックス = log(Z) で決定
  ├─ クラスターキー = (TileX, TileY, ClusterZ)
  │
  ├─ ClusteredLightGrid[ClusterKey] → ライストインデックス一覧取得
  │
  └─ for each LightIndex in クラスターのライスト:
      ├─ GBuffer から BaseColor / Normal / Roughness 等を読み取り
      ├─ FDeferredLightData.Evaluate() → BRDF 計算
      └─ SceneColor に加算書き込み
```

---

## 使用判断

```cpp
// LightRendering.cpp:1581
if (ShouldUseClusteredDeferredShading(ViewFamily.GetShaderPlatform())
    && AreLightsInLightGrid())
{
    StandardDeferredStart = SortedLightSet.ClusteredSupportedEnd;
    bRenderSimpleLightsStandardDeferred = false;
    AddClusteredDeferredShadingPass(...);
}
```

`r.ClusteredDeferredShading` が 0 の場合や、ライトグリッドへの注入に失敗した場合は  
Tiled Deferred または Standard Deferred にフォールバックする。

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.ClusteredDeferredShading` | 1 | Clustered Deferred Shading の有効/無効 |
| `r.LightCulling.Quality` | 2 | ライトカリング品質（0=低/ライトグリッド未使用で Clustered 無効化）|
| `r.Forward.LightGridSizeZ` | 64 | Z 方向のクラスター数 |

---

## 関連リファレンス

- [[ref_clustered_tiled]] — `FClusteredDeferredLightingCS` パラメータ詳細
- [[ref_light_params]] — `FForwardLightData` / `FForwardLightUniformParameters`

---

## ライット割り当て → Cluster グリッド → CS ディスパッチ 詳細フロー

```
【フェーズ 1】ライット割り当て（GatherLightsAndComputeLightGrid）
  │
  ├─ for each FLightSceneInfo:
  │   FForwardLightData を構築
  │   → LightPositionAndInvRadius / LightColorAndFalloff / SpotAngles 等をパック
  │
  ├─ LightDataBuffer（StructuredBuffer<FForwardLightData>）に追加
  │   NumLocalLights = SimpleLightsEnd 〜 ClusteredSupportedEnd の範囲
  │
  └─ BuildLightGrid CS（FLightGridInjectionCS）
      ├─ 各ライストの View Space AABB を計算
      ├─ AABB を 3D クラスターグリッドに投影
      │   GridSize = (TilesX × TilesY × ForwardLightGridSizeZ)
      │   Z 分割: log(ViewZ) で等比数列（奥ほど疎）
      └─ LightLinkedList に (TileX, TileY, ClusterZ) → ライスト数 + インデックス配列

【フェーズ 2】Clustered Deferred Shading CS ディスパッチ
  │
  AddClusteredDeferredShadingPass()
    │
    ├─ [Substrate 対応] for TileType in {Fast, Single, Complex}:
    │   PermutationVector.Set<Substrate::FSubstrateTileType>(TileType)
    │
    ├─ InternalAddClusteredDeferredShadingPass():
    │   FClusteredShadingParameters 構築:
    │     PS.ForwardLightStruct  = View.ForwardLightingResources.ForwardLightUniformBuffer
    │     PS.SceneTextures       = SceneTextures.UniformBuffer
    │     PS.ShadowMaskBits      = VirtualShadowMapMaskBits（なければゼロテクスチャ）
    │     PS.LightFunctionAtlas  = LightFunctionAtlas::BindGlobalParameters()
    │     PS.VirtualShadowMapSamplingParameters = VirtualShadowMapArray.GetSamplingParameters()
    │
    └─ FComputeShaderUtils::AddPass(GraphBuilder, ...)
        DispatchSize = (ceil(Width/8), ceil(Height/8), 1)  ← 8x8 タイル
        → ClusteredDeferredShadingPixelShader.usf 実行

【フェーズ 3】シェーダー内処理（ClusteredDeferredShadingPixelShader.usf）
  │ per-thread (1スレッド = 1ピクセル):
  │
  ├─ Screen UV → クラスターキー計算
  │   TileX = PixelX / 8, TileY = PixelY / 8
  │   ClusterZ = 深度から log 変換でインデックスを取得
  │
  ├─ LightGridBuffer[TileX, TileY, ClusterZ] → ライストインデックス一覧
  │
  ├─ GBuffer デコード（GetGBufferData(ScreenUV)）
  │
  └─ for each LightIndex in Cluster:
      ├─ LightDataBuffer[LightIndex] からライスト情報取得
      ├─ LightingChannelMask チェック（オブジェクトとライストのチャンネルを照合）
      ├─ ShadowMaskBits から VSM シャドウ値取得
      ├─ LightFunctionAtlas から LF 係数取得（有効時）
      ├─ BRDF（GGX Specular + Lambertian Diffuse）評価
      └─ DiffuseColor += ...; SpecularColor += ...
      → SceneColor RT に加算
```
