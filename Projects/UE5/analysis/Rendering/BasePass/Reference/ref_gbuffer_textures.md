# ref: GBuffer テクスチャ（FSceneTextures / FGBufferData）

- 対象: `SceneTextures.h` / `BasePassCommon.ush` / `DeferredShadingCommon.ush`
- Details: [[b_gbuffer_layout]]

---

## FSceneTextures（GBuffer テクスチャ束）

```cpp
// SceneTextures.h
struct FSceneTextures
{
    // --- GBuffer テクスチャ ---
    FRDGTextureMSAA Color;          // SceneColor（HDR）
    FRDGTextureMSAA Depth;          // SceneDepth + Stencil（D24S8）
    FRDGTextureMSAA GBufferA;       // WorldNormal + PerObjectGBufferData
    FRDGTextureMSAA GBufferB;       // Metallic + Specular + Roughness + ShadingModelID
    FRDGTextureMSAA GBufferC;       // BaseColor + AO
    FRDGTextureMSAA GBufferD;       // CustomData（ShadingModel依存）
    FRDGTextureMSAA GBufferE;       // Pre-Shadow Factor（Stationary Light）
    FRDGTextureMSAA GBufferF;       // WorldTangent + Anisotropy（有効時）
    FRDGTextureMSAA Velocity;       // Motion Vector

    // --- その他 ---
    FRDGTextureRef ScreenSpaceAO;
    FRDGTextureRef SmallDepth;      // Downsample Depth（SSAO 等で使用）
    FRDGTextureRef QuarterResDepth;

    // GBuffer RT を配列に束ねるヘルパー
    uint32 GetGBufferRenderTargets(
        TStaticArray<FTextureRenderTargetBinding, MaxSimultaneousRenderTargets>& OutRTs) const;

    // UniformBuffer 生成（シェーダーバインド用）
    TRDGUniformBufferRef<FSceneTextureUniformParameters> UniformBuffer;
};
```

---

## GBuffer チャンネル詳細

| テクスチャ | フォーマット | R | G | B | A |
|----------|------------|---|---|---|---|
| GBufferA | R8G8B8A8_UNORM | WorldNormal.X | WorldNormal.Y | WorldNormal.Z | PerObjectGBufferData |
| GBufferB | R8G8B8A8_UNORM | Metallic | Specular | Roughness | `(ShadingModelID << 4) | SelectiveOutputMask` |
| GBufferC | R8G8B8A8_UNORM | BaseColor.R | BaseColor.G | BaseColor.B | IndirectIrradiance / AO |
| GBufferD | R8G8B8A8_UNORM | CustomData 4 ch（ShadingModel依存）| | | |
| GBufferE | R8G8B8A8_UNORM | PreShadow0 | PreShadow1 | PreShadow2 | PreShadow3 |
| GBufferF | R8G8B8A8_UNORM | WorldTangent.X | WorldTangent.Y | WorldTangent.Z | Anisotropy |
| Velocity | R16G16_FLOAT | MotionVec.X | MotionVec.Y | — | — |

---

## FGBufferData（CPU/シェーダー側デコード構造体）

```cpp
// DeferredShadingCommon.ush（HLSL）/ SceneRendering.h 相当
struct FGBufferData
{
    // --- デコード済み値 ---
    float3 WorldNormal;       // ワールド法線（単位ベクトル）
    float  PerObjectGBufferData;
    float  Metallic;
    float  Specular;          // スペキュラー強度（デフォルト 0.5）
    float  Roughness;         // ラフネス（PBR）
    float  Material;          // 旧 SubsurfaceProfile Index
    uint   ShadingModelID;    // EMaterialShadingModel のインデックス
    uint   SelectiveOutputMask;
    float3 BaseColor;         // アルベド
    float  AO;                // アンビエントオクルージョン [0,1]
    float  IndirectIrradiance;
    float4 CustomData;        // ShadingModel 別の追加データ
    float3 PrecomputedShadowFactors; // Stationary Light プリシャドウ
    float3 WorldTangent;      // 異方性用タンジェント
    float  Anisotropy;        // 異方性強度 [-1,1]
};
```

---

## HLSL でのデコード（DeferredShadingCommon.ush）

```hlsl
// GBuffer からデコードするメイン関数
FGBufferData GetGBufferData(float2 UV, bool bGetNormalizedNormal = true)
{
    FGBufferData GBuffer;
    float4 BufferA = GBufferATexture.SampleLevel(GBufferATextureSampler, UV, 0);
    float4 BufferB = GBufferBTexture.SampleLevel(GBufferBTextureSampler, UV, 0);
    float4 BufferC = GBufferCTexture.SampleLevel(GBufferCTextureSampler, UV, 0);
    float4 BufferD = GBufferDTexture.SampleLevel(GBufferDTextureSampler, UV, 0);

    GBuffer.WorldNormal    = BufferA.rgb * 2.0f - 1.0f; // oct or direct encode
    GBuffer.Metallic       = BufferB.r;
    GBuffer.Specular       = BufferB.g;
    GBuffer.Roughness      = BufferB.b;
    GBuffer.ShadingModelID = (uint)(BufferB.a * 15.999f);
    GBuffer.BaseColor      = BufferC.rgb;
    GBuffer.AO             = BufferC.a;
    GBuffer.CustomData     = BufferD;
    return GBuffer;
}
```

---

> [!note]- GBufferB.A のパック構造
> `GBufferB.A = (ShadingModelID * (1.0/16.0)) + (SelectiveOutputMask * (1.0/256.0))`
> ShadingModelID は上位4bit、SelectiveOutputMask は下位4bit にパックされる。
> デコード: `ShadingModelID = (uint)(GBufferB.a * 15.999f)`

> [!note]- SelectiveBasePassOutputs（r.SelectiveBasePassOutputs）
> `r.SelectiveBasePassOutputs=1` の時、シェーダーは全 RT ではなく実際に値を書き込む RT のみに出力する。
> 例: Unlit マテリアルは GBufferA/B/D への書き込みをスキップして帯域を節約できる。
> **シェーダーリコンパイルが必要**（ECVF_ReadOnly）。
