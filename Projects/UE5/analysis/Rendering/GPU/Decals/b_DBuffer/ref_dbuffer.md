# REF: DBuffer デカール シェーダー

- グループ: b - DBuffer
- 詳細: [[detail_dbuffer]]
- CPU リファレンス: [[ref_dbuffer]]
- ソース: `Engine/Shaders/Private/DeferredDecal.usf`  
          `Engine/Shaders/Private/DBufferCommon.ush`

---

## FPixelShaderInOut_MainPS（DBuffer モード）

DeferredDecal.usf の `FPixelShaderInOut_MainPS()` は Deferred Decal と同じ関数を使用するが、  
`DECAL_RENDERTARGETMODE = DECAL_RENDERTARGETMODE_DBUFFER` パーミュテーションにより  
DBuffer への書き込みに切り替わる。

### パーミュテーション（DBuffer 固有）

| マクロ | 値 | 説明 |
|-------|------|------|
| `DECAL_RENDERTARGETMODE` | `DECAL_RENDERTARGETMODE_DBUFFER` | DBufferA/B/C に書き込み |
| `DECAL_RENDERSTAGE` | `DECAL_RENDERSTAGE_BEFOREBASEPASS` | BasePass 前に実行 |

---

## DBufferCommon.ush 主要定義

### FDBufferData 構造体

```hlsl
struct FDBufferData
{
    float4 ColorOpacity;                     // rgb=BaseColor, a=Opacity
    float4 NormalOpacity;                    // rgb=Normal（-1〜1）, a=Opacity
    float  RoughnessMetallicSpecularOpacity; // DBufferC の Lerp 係数
    float3 RoughnessMetallicSpecular;        // r=Roughness, g=Metallic, b=Specular
};
```

### DecodeGBufferDataDBuffer（GetDBufferData）

```hlsl
FDBufferData GetDBufferData(float2 ScreenUV, uint ShadingModelMask)
{
    FDBufferData DBufferData;
    float4 DBufferA_Sample = DBufferATexture.SampleLevel(DBufferATextureSampler, ScreenUV, 0);
    float4 DBufferB_Sample = DBufferBTexture.SampleLevel(DBufferBTextureSampler, ScreenUV, 0);
    float4 DBufferC_Sample = DBufferCTexture.SampleLevel(DBufferCTextureSampler, ScreenUV, 0);

    DBufferData.ColorOpacity = DBufferA_Sample;
    DBufferData.NormalOpacity = float4(DBufferB_Sample.xyz * 2 - 1, DBufferB_Sample.w);
    DBufferData.RoughnessMetallicSpecular = DBufferC_Sample.rgb;
    DBufferData.RoughnessMetallicSpecularOpacity = DBufferC_Sample.a;
    return DBufferData;
}
```

### ApplyDBufferData

```hlsl
void ApplyDBufferData(
    FDBufferData DBufferData,
    inout half3 WorldNormal,
    inout half3 SubsurfaceColor,
    inout half Roughness,
    inout half3 BaseColor,
    inout half Metallic,
    inout half Specular)
// BasePass のマテリアルシェーダーから呼ばれる
// Lerp 係数（各テクスチャの Alpha）で既存値とデカール値をブレンド
```

---

## FDBufferTextures（CPU 側）

```cpp
// DecalRenderingCommon.h
struct FDBufferTextures
{
    FRDGTextureRef DBufferA = nullptr;  // BaseColor + Opacity
    FRDGTextureRef DBufferB = nullptr;  // Normal + Opacity
    FRDGTextureRef DBufferC = nullptr;  // Roughness/Metallic/Specular + Opacity

    bool IsValid() const { return DBufferA && DBufferB && DBufferC; }
};

// RenderDeferredDecals() で確保・BasePass にバインド
```

---

> [!note]- DBuffer の Lerp 係数の意味
> DBufferA/B/C の Alpha チャンネルは「どれだけデカールで上書きするか」の係数。  
> Alpha = 0 ならデカールなし（元の GBuffer 値を使用）、Alpha = 1 ならデカールで完全上書き。  
> 複数のデカールが重なる場合、後から描画されるデカールが `lerp(前のDBuffer値, 新しい値, Opacity)` で合算される（Premultiplied Alpha）。
