# REF: DiffuseIndirectComposite.usf シェーダー

- グループ: g - Final Composite
- 詳細: [[detail_diffuse_indirect_composite]]
- CPU リファレンス: [[ref_lumen_final_composite]]
- ソース: `Engine/Shaders/Private/DiffuseIndirectComposite.usf`

---

## エントリポイント

### MainPS

```hlsl
void MainPS(
    float4 SvPosition : SV_POSITION
#if DIM_APPLY_DIFFUSE_INDIRECT
  #if ENABLE_DUAL_SRC_BLENDING
    , out float4 OutAddColor      DUAL_SOURCE_BLENDING_SLOT(0) : SV_Target0
    , out float4 OutMultiplyColor DUAL_SOURCE_BLENDING_SLOT(1) : SV_Target1
  #else
    , out float4 OutColor                                       : SV_Target0
  #endif
#else
    , out float4 OutMultiplyColor : SV_Target0
#endif
)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Pixel Shader |
| **目的** | Diffuse GI + 反射 + AO を GBuffer と合成して SceneColor に書き込む |
| **RT** | SceneColor（加算ブレンドまたは Dual Source Blending）|
| **CPU クラス** | `FDiffuseIndirectCompositePS`（`IndirectLightRendering.cpp:119`）|

---

## シェーダーパーミュテーション一覧

| パーミュテーションクラス | マクロ名 | 値 | 説明 |
|---------------------|---------|-----|------|
| `FApplyDiffuseIndirectDim` | `DIM_APPLY_DIFFUSE_INDIRECT` | 0/1/2 | 0=なし, 1=SSGI/Plugin, 2=Lumen |
| `FUpscaleDiffuseIndirectDim` | `DIM_UPSCALE_DIFFUSE_INDIRECT` | 0/1 | SSGI のみ有効 |
| `FScreenBentNormalMode` | `DIM_SCREEN_BENT_NORMAL_MODE` | 0/1/2 | 0=なし, 1=SSBN, 2=ShortRangeAO |
| `FShortRangeGI` | `DIM_SHORT_RANGE_GI` | 0/1 | Lumen 近距離 GI 有効化 |
| `FEnableDualSrcBlending` | `ENABLE_DUAL_SRC_BLENDING` | 0/1 | Dual Source Blending 対応 GPU |
| `FHairComplexTransmittance` | `USE_HAIR_COMPLEX_TRANSMITTANCE` | 0/1 | Hair Strands 透過マテリアル対応 |
| `FScreenSpaceReflectionTiledComposition` | `SSR_TILED_COMPOSITION` | 0/1 | SSR タイル合成（Lumen + SSR 混在）|
| `Substrate::FSubstrateTileType` | — | 0/1/2 | Substrate タイルタイプ（Simple/SLW/Complex）|

### コンパイル除外ルール（ShouldCompilePermutation）

- モバイルプラットフォームは全パーミュテーション除外
- `DIM_APPLY_DIFFUSE_INDIRECT != 2` のとき `DIM_SCREEN_BENT_NORMAL_MODE != 0` または `DIM_SHORT_RANGE_GI` は除外
- `DIM_APPLY_DIFFUSE_INDIRECT != 1` のとき `DIM_UPSCALE_DIFFUSE_INDIRECT` は除外
- Substrate タイルパーミュテーションは `DIM_APPLY_DIFFUSE_INDIRECT == 2` のみ

---

## 主要シェーダーパラメーター

| パラメーター | 型 | 内容 |
|------------|-----|------|
| `DiffuseIndirect_Lumen_0` | `Texture2DArray` | Diffuse GI メイン（Screen Probe Gather）|
| `DiffuseIndirect_Lumen_1` | `Texture2DArray` | Backface Diffuse |
| `DiffuseIndirect_Lumen_2` | `Texture2DArray` | Rough Specular Indirect |
| `DiffuseIndirect_Lumen_3` | `Texture2DArray` | Reflections（Lumen or SSR）|
| `AmbientOcclusionTexture` | `Texture2D` | SSAO / RTAO / Bent Normal AO スカラー |
| `AmbientOcclusionStaticFraction` | `float` | 静的 AO のブレンド率（PostProcess Volume から）|
| `PreIntegratedGF` | `Texture2D` | BRDF 鏡面積分テーブル（GF = Fresnel の事前積分）|
| `LumenReflectionSpecularScale` | `float` | 反射の明るさスケール |
| `LumenReflectionContrast` | `float` | 反射のコントラスト調整 |
| `bLumenSupportBackfaceDiffuse` | `uint` | Backface Diffuse を使うか |
| `bLumenReflectionInputIsSSR` | `uint` | Lumen_3 が SSR 入力か（Lumen 反射代わり）|
| `SceneTexturesStruct` | `FSceneTextureUniformParameters` | GBuffer 参照用 |
| `ViewUniformBuffer` | `FViewUniformShaderParameters` | View 変換行列等 |
| `OutOpaqueRoughRefractionSceneColor` | `RWTexture2D<float3>` | Substrate 不透明粗面屈折出力（UAV）|
| `OutSubSurfaceSceneColor` | `RWTexture2D<float3>` | Substrate サブサーフェス分離出力（UAV）|

---

## DiffuseIndirect_Lumen 系テクスチャのスロット割り当て

| スロット | テクスチャ名 | CPU 側割り当て元 |
|---------|------------|----------------|
| `Textures[0]` | `DiffuseIndirect_Lumen_0` | `RenderLumenFinalGather()` の返値 — メイン Diffuse GI |
| `Textures[1]` | `DiffuseIndirect_Lumen_1` | 同上 — Backface Diffuse（Two-sided マテリアル用）|
| `Textures[2]` | `DiffuseIndirect_Lumen_2` | 同上 — Rough Specular Indirect（粗面反射 GI）|
| `Textures[3]` | `DiffuseIndirect_Lumen_3` | `RenderLumenReflections()` の返値 — 反射バッファ |

---

## 出力ターゲット

| 名前 | 内容 |
|-----|------|
| `OutAddColor`（Dual Src Blending）| SceneColor に加算する輝度 `float4(GI + Specular, 0)` |
| `OutMultiplyColor`（Dual Src Blending）| SceneColor に乗算するマスク `float4(AO, AO, AO, 1)` |
| `OutColor`（非対応 GPU）| SceneColor コピーに GI を合成した最終色 |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.DiffuseIndirectForceCopyPass` | 0 | Dual Source Blending を強制的に無効化してコピーパスを使用 |
| `r.Lumen.DiffuseIndirect.Visualize` | 0 | GI 合成前テクスチャのデバッグ表示（`bVisualizeDiffuseIndirect`）|

---

> [!note]- Dual Source Blending の仕組みと恩恵
> `OutAddColor` と `OutMultiplyColor` を同時に出力できる GPU では、SceneColor を読み取らずに1回のドローコールで  
> `SceneColor_final = SceneColor_current * OutMultiplyColor.rgb + OutAddColor.rgb`  
> が計算できる。これは AO（乗算）と GI 加算を1パスで完結させる最適化。  
> 非対応の場合は `AddCopyTexturePass()` でコピーしてから `PSParameters->SceneColorTexture` に渡す（追加パスが必要）。

> [!note]- Backface Diffuse（Textures[1]）の用途
> `bLumenSupportBackfaceDiffuse` が true のとき、`Textures[1]` には Two-sided マテリアルや薄いオブジェクト（葉など）の**裏面 GI** が入る。  
> `DiffuseIndirect_Lumen_1` の輝度は表面の BackfaceDiffuse マスクに乗算されて加算されるため、木の葉が透過して輝いて見える効果が得られる。

> [!note]- Rough Specular と純粋な反射（Textures[2] vs [3]）の違い
> `Textures[2]`（RoughSpecularIndirect）は Screen Probe Gather のトレース結果から Roughness 大のピクセル向けに補間された「粗面専用の GI」。  
> `Textures[3]`（Reflections）は Lumen Reflection パスで独立してトレース・デノイズされた「高精度な反射」（Roughness 小向け）。  
> DiffuseIndirectComposite はこれらを Roughness で重み付けして BRDF に合わせて合成する。
