# REF: Lumen Screen Space Short Range AO シェーダー

- グループ: b - Lumen AO
- 詳細: [[detail_lumen_ao]]
- CPU リファレンス: [[ref_composition_lighting]]
- ソース: `Engine/Shaders/Private/Lumen/LumenScreenSpaceBentNormal.usf`

---

## ScreenSpaceShortRangeAOCS（LumenScreenSpaceBentNormal.usf:637）

### エントリポイント

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ScreenSpaceShortRangeAOCS(
    uint2 PixelPos : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Compute Shader |
| **スレッドグループ** | 8×8（THREADGROUP_SIZE=8）|
| **目的** | 短距離スクリーンスペース AO + Bent Normal を計算 |
| **出力** | `RWBentNormalTexture`（float4: xyz=BentNormal, w=AO）|

#### 入力バインド

| バインド名 | 型 | 説明 |
|-----------|------|------|
| `SceneDepthTexture` | `Texture2D` | シーン深度 |
| `GBufferATexture` | `Texture2D` | GBuffer A（法線）|
| `LumenAOParameters` | `UniformBuffer` | サンプル数・半径・フェード設定 |

#### 出力バインド

| バインド名 | 型 | 説明 |
|-----------|------|------|
| `RWBentNormalTexture` | `RWTexture2D<float4>` | xyz=BentNormal（-1〜1 を 0〜1 にエンコード）, w=AO |

#### CPU バインド

```cpp
// IndirectLightRendering.cpp
FLumenScreenSpaceBentNormalParameters* PassParameters = ...;
PassParameters->SceneDepthTexture = SceneTextures.SceneDepthTexture;
PassParameters->GBufferATexture = SceneTextures.GBufferATexture;
PassParameters->RWBentNormalTexture = GraphBuilder.CreateUAV(BentNormalTexture);

FComputeShaderUtils::AddPass(
    GraphBuilder,
    RDG_EVENT_NAME("LumenScreenSpaceShortRangeAO"),
    ComputeShader,
    PassParameters,
    FComputeShaderUtils::GetGroupCount(View.ViewRect.Size(), THREADGROUP_SIZE));
```

---

## 関連 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Lumen.ScreenBentNormal` | 1 | Screen Space Bent Normal AO 有効 |
| `r.Lumen.ShortRangeAO.NumSteps` | 4 | サンプルステップ数 |
| `r.Lumen.ShortRangeAO.Radius` | 0.05 | AO サンプリング半径（スクリーン割合）|
| `r.Lumen.ShortRangeAO.MaxPixelRadius` | 200 | 最大ピクセル半径 |

---

> [!note]- Lumen AO vs GTAO の使い分け
> - **Lumen 有効時**: `ScreenSpaceShortRangeAOCS` が動作し、GTAO/SSAO はスキップされる  
> - **Lumen 無効時**: `RenderGTAO()` または `FRCPassPostProcessAmbientOcclusion` が動作  
> - 判定は `ProcessAfterOcclusion()` 内の `bLumenActive` フラグで行われる
