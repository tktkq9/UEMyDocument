# REF: Depth Only シェーダー

- グループ: a - DepthOnly
- 詳細: [[detail_depth_only]]
- CPU リファレンス: [[ref_depth_rendering]]
- ソース: `Engine/Shaders/Private/DepthOnlyVertexShader.usf`  
          `Engine/Shaders/Private/DepthOnlyPixelShader.usf`

---

## DepthOnlyVertexShader.usf#Main（:32）

### エントリポイント

```hlsl
void Main(
    FVertexFactoryInput Input,
    out FDepthOnlyVSOutput Output
#if USE_GLOBAL_CLIP_PLANE
    , out float OutGlobalClipPlaneDistance : SV_ClipDistance
#endif
    , out FStereoVSOutput StereoOutput)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Vertex Shader |
| **目的** | メッシュをクリップ空間に変換（WPO 適用）。深度のみ書き込み |
| **ストリーム** | Opaque: `EVertexInputStreamType::PositionOnly`（位置のみ）|
|              | Masked: `EVertexInputStreamType::Default`（フルストリーム）|

#### FDepthOnlyVSToPS

```hlsl
struct FDepthOnlyVSToPS
{
    INVARIANT_OUTPUT float4 Position : SV_POSITION;
    // MATERIALBLENDING_SOLID == false 時のみ以下を出力:
    FVertexFactoryInterpolantsVSToPS FactoryInterpolants;
    float4 PixelPosition : TEXCOORD6;
    // USE_RAW_WORLD_POSITION 時:
    float3 PixelPositionExcludingWPO : TEXCOORD7;
};
```

#### パーミュテーション

| マクロ | 説明 |
|-------|------|
| `MATERIALBLENDING_SOLID` | Solid ブレンド = PS なし・位置ストリーム |
| `OUTPUT_PIXEL_DEPTH_OFFSET` | PDO 対応（PixelDepthOffset マテリアル）|
| `USE_WORLD_POSITION_EXCLUDING_SHADER_OFFSETS` | WPO 除外ワールド座標を出力 |

---

## DepthOnlyPixelShader.usf#Main（:15）

### エントリポイント

```hlsl
void Main(
#if !MATERIALBLENDING_SOLID || OUTPUT_PIXEL_DEPTH_OFFSET
    in INPUT_POSITION_QUALIFIERS float4 SvPosition : SV_Position,
    FVertexFactoryInterpolantsVSToPS FactoryInterpolants,
    float4 PixelPosition : TEXCOORD6,
    in FStereoPSInput StereoInput,
    OPTIONAL_IsFrontFace
    OPTIONAL_OutDepthConservative,
#endif
    out float4 OutColor : SV_Target0
#if MATERIALBLENDING_MASKED_USING_COVERAGE
    , out uint OutCoverage : SV_Coverage
#endif
    )
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Pixel Shader |
| **コンパイル条件** | Masked または PDO マテリアルのみ |
| **Opacity テスト** | `GetMaterialClippingPS()` で `clip(Opacity - 0.333)` |
| **PDO 出力** | `SV_DepthGreaterEqual` で深度値を出力（通常より手前に引き寄せ）|

---

## FDepthPassMeshProcessor（DepthRendering.cpp）

```cpp
// CPU 側: Depth Pass メッシュコマンド構築
bool FDepthPassMeshProcessor::TryAddMeshBatch(
    const FMeshBatch& Mesh,
    uint64 BatchElementMask,
    const FPrimitiveSceneProxy* PrimitiveSceneProxy,
    int32 StaticMeshId,
    const FMaterialRenderProxy& MaterialRenderProxy,
    const FMaterial& Material)
{
    // bUsePositionOnlyStream = (Material.WritesEveryPixel() && !bSupportPositionOnlyStream==false)
    bool bUsePositionOnlyStream = !Material.IsMasked() && !Material.IsTranslucent();

    if (bUsePositionOnlyStream)
    {
        // 軽量 PositionOnly VS
        Process<true>(Mesh, ...);
    }
    else
    {
        // フル VS + PS
        Process<false>(Mesh, ...);
    }
}
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.EarlyZPass` | 3 | 0=無効/1=Masked のみ/2=Opaque のみ/3=全て |
| `r.EarlyZPassMovable` | 0 | 動的メッシュも Pre-pass に含める |
| `r.EarlyZPassOnlyMaterialMasking` | 0 | マテリアルマスキングのみ EarlyZ |
