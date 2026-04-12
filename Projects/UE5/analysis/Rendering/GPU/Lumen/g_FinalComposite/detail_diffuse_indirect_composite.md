# Lumen DiffuseIndirectComposite シェーダー詳細

- グループ: g - Final Composite
- GPU 概要: [[01_gpu_overview]]
- CPU 詳細: [[g_lumen_final_composite]]
- リファレンス: [[ref_diffuse_indirect_composite]]

---

## 概要

Lumen パイプラインの **最終段**。  
Screen Probe Gather（Diffuse GI）・Reflections・AO の各テクスチャを GBuffer と合成して SceneColor に書き込む。  
Pixel Shader（`MainPS`）として実行され、RenderTarget は SceneColor そのもの。

---

## シェーダーの役割

```
入力テクスチャ群
  ├─ DiffuseIndirect_Lumen_0    … Diffuse GI（Screen Probe Gather メイン）
  ├─ DiffuseIndirect_Lumen_1    … Backface Diffuse（植生・薄い壁の裏面 GI）
  ├─ DiffuseIndirect_Lumen_2    … Rough Specular Indirect（粗面反射 GI）
  ├─ DiffuseIndirect_Lumen_3    … Reflections（Lumen 反射 or SSR）
  └─ AmbientOcclusionTexture     … SSAO / RTAO / Lumen Bent Normal AO
          ↓
  GBuffer（マテリアルデータ）
  ├─ DiffuseColor, SpecularColor, Roughness, Normal
  ├─ BRDF 評価（FDirectLighting IndirectLighting）
  └─ PreIntegratedGF（鏡面反射の事前積分テーブル）
          ↓
  SceneColor への加算書き込み
  ├─ Dual Source Blending 対応時: OutAddColor + OutMultiplyColor（1パス）
  └─ 非対応時: SceneColorTexture コピー後に OutColor として出力
```

---

## MainPS のコアロジック（DIM_APPLY_DIFFUSE_INDIRECT == 2 = Lumen 時）

```hlsl
void MainPS(float4 SvPosition : SV_POSITION,
    out float4 OutAddColor : SV_Target0,       // Dual Source Blending 時
    out float4 OutMultiplyColor : SV_Target1)  // Dual Source Blending 時
{
    // 1. GBuffer からマテリアルデータを読み取り
    FLumenMaterialData Material = ReadMaterialData(PixelPos, SceneBufferUV);

    // 2. AO の計算（LumenGI 使用時は FinalAmbientOcclusion = 1.0、AO は内部処理）
    float DynamicAmbientOcclusion = AmbientOcclusionTexture.Sample(...);
    float FinalAmbientOcclusion = 1.0f;  // Lumen パスでは AO はシェーダー内部処理

    // 3. Diffuse GI の取得（Texture2DArray からサンプリング）
    DiffuseIndirectLighting   = DiffuseIndirect_Lumen_0.Sample(…, float3(UV, 0));  // Diffuse
    RoughSpecularLighting     = DiffuseIndirect_Lumen_2.Sample(…, float3(UV, 0));  // Rough Specular
    SpecularIndirectLighting  = DiffuseIndirect_Lumen_3.Sample(…, float3(UV, 0));  // Reflections

    // 4. Screen Bent Normal AO の適用（DIM_SCREEN_BENT_NORMAL_MODE）
    // Bent Normal 方向で GI をバイアス → Light Leaking を抑制

    // 5. BRDF × GI の計算
    IndirectLighting.Diffuse   = DiffuseIndirectLighting * Material.DiffuseColor * AOMultibounce;
    IndirectLighting.Specular  = SpecularIndirectLighting.rgb * EnvBRDF(PreIntegratedGF, Roughness, NoV);

    // 6. Backface Diffuse（DIM_APPLY_DIFFUSE_INDIRECT == 2 かつ bLumenSupportBackfaceDiffuse）
    BackfaceDiffuse = DiffuseIndirect_Lumen_1.Sample(…, float3(UV, 0));
    IndirectLighting.Diffuse += BackfaceDiffuse * BackfaceDiffuseWeight;

    // 7. SceneColor への書き込み
    OutAddColor      = float4(TotalIndirectLighting, 0);
    OutMultiplyColor = float4(AOMultiply, AOMultiply, AOMultiply, 1.0);
    // SceneColor_final = SceneColor_current * OutMultiplyColor + OutAddColor
}
```

---

## パーミュテーション × 実行パス

| `DIM_APPLY_DIFFUSE_INDIRECT` | 動作 |
|-----|------|
| 0 | 合成なし（AO のみ乗算）|
| 1 | SSGI / Plugin（単一入力テクスチャ）|
| 2 | **Lumen**（Screen Probe Gather + 反射 + Bent Normal）|

| `DIM_SCREEN_BENT_NORMAL_MODE` | 動作 |
|-----|------|
| 0 | Bent Normal なし |
| 1 | Screen Space Bent Normal を適用 |
| 2 | Short Range AO（近距離 HW RT）を適用 |

| `ENABLE_DUAL_SRC_BLENDING` | 動作 |
|-----|------|
| true | 1パスで Add + Multiply を同時出力（コピーパス不要）|
| false | `SceneColorTexture` コピー後に単一 OutColor で出力 |

---

## Substrate タイル別実行

Substrate が有効の場合、CPU 側は `ApplyDiffuseIndirect` ラムダを  
タイルタイプ別（Simple / SingleLayerWater / Complex）に 3 回呼ぶ。

```
ApplyDiffuseIndirect(ESubstrateTileType::Simple)         ← 単純マテリアル（低コスト）
ApplyDiffuseIndirect(ESubstrateTileType::SingleLayerWater) ← 水面
ApplyDiffuseIndirect(ESubstrateTileType::Complex)        ← GGX 等複雑マテリアル
```

各タイルは `Substrate::FSubstrateTilePassVS` でタイルマスクを適用し、  
関係しないピクセルをスキップして余分なシェーディングコストを削減する。

---

## CPU 呼び出しの流れ

```
RenderDiffuseIndirectAndAmbientOcclusion()   // IndirectLightRendering.cpp:977
  │
  └─ ApplyDiffuseIndirect ラムダ × 3（Substrate タイル数）
       │
       ├─ GraphBuilder.AllocParameters<FDiffuseIndirectCompositeParameters>()
       ├─ PS.DiffuseIndirect_Lumen_0 = FSSDSignalTextures::Textures[0]  (Diffuse GI)
       ├─ PS.DiffuseIndirect_Lumen_1 = FSSDSignalTextures::Textures[1]  (Backface)
       ├─ PS.DiffuseIndirect_Lumen_2 = FSSDSignalTextures::Textures[2]  (Rough Specular)
       ├─ PS.DiffuseIndirect_Lumen_3 = FSSDSignalTextures::Textures[3]  (Reflections)
       ├─ PS.AmbientOcclusionTexture = AmbientOcclusionMask
       ├─ PS.PreIntegratedGF         = GSystemTextures.PreintegratedGF
       ├─ RenderTargets[0]           = SceneColor（AddColor）
       └─ GraphBuilder.AddPass(FDiffuseIndirectCompositePS)
```
