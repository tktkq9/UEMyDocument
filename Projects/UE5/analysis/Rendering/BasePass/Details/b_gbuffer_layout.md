# b: GBuffer レイアウト

- 対象ファイル: `BasePassCommon.ush` / `SceneTextures.h` / `GBufferInfo.h`
- 概要: [[14_basepass_overview]]

---

## 概要

**GBuffer（Geometry Buffer）** は Deferred Rendering の中核データ構造。  
BasePass でマテリアル評価結果を複数のテクスチャに格納し、  
後続の Lighting Pass がそこから読み取って照明計算を行う。

---

## GBuffer レイアウト（従来 Shading Model）

| スロット | フォーマット | R | G | B | A |
|--------|------------|---|---|---|---|
| **GBufferA** | R8G8B8A8_UNORM | WorldNormal.X | WorldNormal.Y | WorldNormal.Z | PerObjectGBufferData |
| **GBufferB** | R8G8B8A8_UNORM | Metallic | Specular | Roughness | ShadingModelID（4bit）+ SelectiveOutputMask（4bit）|
| **GBufferC** | R8G8B8A8_UNORM | BaseColor.R | BaseColor.G | BaseColor.B | GBufferAO / IndirectIrradiance |
| **GBufferD** | R8G8B8A8_UNORM | CustomData.X | CustomData.Y | CustomData.Z | CustomData.W（ShadingModel依存）|
| **GBufferE** | R8G8B8A8_UNORM | Pre-computed shadow factor（Stationary Light 4チャンネル）| | | |
| **GBufferF** | R8G8B8A8_UNORM | WorldTangent.X | WorldTangent.Y | WorldTangent.Z | Anisotropy（Anisotropy 有効時のみ）|
| **SceneDepth** | D24S8 | — | — | — | Stencil: ShadingModelID |
| **Velocity** | R16G16_FLOAT | MotionVector.X | MotionVector.Y | — | — |

---

## ShadingModelID（GBufferB.A 上位4bit）

```cpp
// ShadingModels.h
enum EMaterialShadingModel : uint8
{
    MSM_Unlit             = 0,   // GBuffer 不要
    MSM_DefaultLit        = 1,   // Lambertian + GGX
    MSM_Subsurface        = 2,   // サブサーフェス（CustomData: SubsurfaceColor）
    MSM_PreintegratedSkin = 3,   // プリインテグレーテッドスキン
    MSM_ClearCoat         = 4,   // クリアコート（CustomData: ClearCoat + ClearCoatRoughness）
    MSM_SubsurfaceProfile = 5,   // SSS プロファイル参照
    MSM_TwoSidedFoliage   = 6,   // 両面植生
    MSM_Hair              = 7,   // Hair Strands（Kajiya-Kay）
    MSM_Cloth             = 8,   // 布（CustomData: FuzzColor）
    MSM_Eye               = 9,   // 眼球（CustomData: IrisMask + IrisDistance）
    MSM_SingleLayerWater  = 11,  // 単層水面
    MSM_ThinTranslucent   = 12,  // 薄い半透明
    // ...
};
```

---

## CustomData（GBufferD）の ShadingModel 別解釈

| ShadingModel | GBufferD.R | GBufferD.G | GBufferD.B | GBufferD.A |
|-------------|-----------|-----------|-----------|-----------|
| DefaultLit | — | — | — | — |
| Subsurface | SubsurfaceColor.R | SubsurfaceColor.G | SubsurfaceColor.B | Opacity |
| ClearCoat | ClearCoat | ClearCoatRoughness | — | — |
| Hair | Backlit | — | — | — |
| Cloth | FuzzColor.R | FuzzColor.G | FuzzColor.B | Cloth |
| Eye | IrisMask | IrisDistance | — | — |

---

## Substrate 有効時の差異

`r.Substrate=1` の場合、GBufferA/B/C/D/E/F は使用されず  
`FSubstrateSceneData::MaterialTextureArray`（Texture2DArray uint）に置き換わる。

```cpp
// タイル分類ステンシルビット（Substrate）
constexpr uint32 StencilBit_Fast           = 0x10; // 単純マテリアル
constexpr uint32 StencilBit_Single         = 0x20; // 中程度
constexpr uint32 StencilBit_Complex        = 0x40; // 複数クロージャ
constexpr uint32 StencilBit_ComplexSpecial = 0x80; // 特殊（Eye 等）
```

---

## GBuffer の生成〜バインド〜デコードフロー

```
[BasePass]
マテリアルシェーダー (BasePassPixelShader.usf)
  └─ EncodeGBuffer(GBufferData)
      └─ 各チャンネルに圧縮エンコードして RT に書き込み

[Lighting Pass]
DeferredLightPixelShaders.usf
  └─ GetGBufferData(ScreenUV)
      └─ DecodeGBuffer()
          ├─ WorldNormal = GBufferA.xyz * 2 - 1 (oct encode)
          ├─ Metallic = GBufferB.r
          ├─ Specular = GBufferB.g
          ├─ Roughness = GBufferB.b
          ├─ ShadingModelID = (uint)(GBufferB.a * 15.0 + 0.5)
          └─ BaseColor = GBufferC.rgb
```

---

## 関連リファレンス

- [[ref_gbuffer_textures]] — `FSceneTextures` / `FGBufferData` 全フィールド
- [[ref_basepass_common]] — `BasePassCommon.ush` エンコード定数

---

## GBuffer テクスチャ生成〜バインド〜デコード 詳細フロー

```
【生成フェーズ】FSceneTextures::InitializeViewFamily()
  │ SceneTextures.h/.cpp
  │
  ├─ CreateRenderTargetDesc() で各 GBuffer テクスチャの仕様を決定
  │   フォーマット: r.GBufferFormat (0=Force8Bit / 1=Default / 3=HighPrecision)
  │   MSAA サンプル数: r.MSAA.NumSamples
  │
  ├─ GBufferA = RDGBuilder.CreateTexture(Desc_R8G8B8A8, "GBufferA")
  ├─ GBufferB = RDGBuilder.CreateTexture(Desc_R8G8B8A8, "GBufferB")
  ├─ GBufferC = RDGBuilder.CreateTexture(Desc_R8G8B8A8, "GBufferC")
  ├─ GBufferD = RDGBuilder.CreateTexture(Desc_R8G8B8A8, "GBufferD")
  ├─ GBufferE = RDGBuilder.CreateTexture(Desc_R8G8B8A8, "GBufferE")  ← Stationary Light あり時
  ├─ GBufferF = RDGBuilder.CreateTexture(Desc_R8G8B8A8, "GBufferF")  ← Anisotropy 有効時
  └─ Velocity = RDGBuilder.CreateTexture(Desc_R16G16_FLOAT, "Velocity")

【バインドフェーズ】GetGBufferRenderTargets()
  │ BasePassRendering.cpp から呼び出し
  │
  └─ TStaticArray<FTextureRenderTargetBinding> OutRTs:
      [0] SceneColor  (RT0)
      [1] GBufferA    (RT1)
      [2] GBufferB    (RT2)
      [3] GBufferC    (RT3)
      [4] GBufferD    (RT4)
      [5] GBufferE    (RT5, 有効時)
      [6] GBufferF    (RT6, 有効時)
      [7] Velocity    (RT7)
      DepthStencil    (DSV: SceneDepth D24S8)

【書き込みフェーズ】BasePassPixelShader.usf
  │
  └─ FPixelShaderOut.MRT[0..N]:
      EncodeGBuffer(FGBufferData GBuffer, out float4 OutGBufferA, ...):
        OutGBufferA.xyz = GBuffer.WorldNormal * 0.5 + 0.5   // [0,1] に正規化
        OutGBufferA.w   = GBuffer.PerObjectGBufferData
        OutGBufferB.x   = GBuffer.Metallic
        OutGBufferB.y   = GBuffer.Specular
        OutGBufferB.z   = GBuffer.Roughness
        OutGBufferB.w   = (ShadingModelID * (1/16.0)) + (SelectiveMask * (1/256.0))
        OutGBufferC.xyz = GBuffer.BaseColor
        OutGBufferC.w   = GBuffer.IndirectIrradiance (or AO)

【デコードフェーズ】DeferredShadingCommon.ush
  │ Lighting Pass / MegaLights Resolve などから呼び出し
  │
  └─ GetGBufferData(float2 UV):
      float4 A = GBufferATexture.SampleLevel(sampler, UV, 0)
      float4 B = GBufferBTexture.SampleLevel(sampler, UV, 0)
      float4 C = GBufferCTexture.SampleLevel(sampler, UV, 0)
      float4 D = GBufferDTexture.SampleLevel(sampler, UV, 0)

      GBuffer.WorldNormal    = A.xyz * 2.0 - 1.0   // [0,1] → [-1,1]
      GBuffer.Metallic       = B.x
      GBuffer.Specular       = B.y
      GBuffer.Roughness      = B.z
      GBuffer.ShadingModelID = (uint)(B.w * 15.999) // 上位4bit を復元
      GBuffer.BaseColor      = C.xyz
      GBuffer.AO             = C.w
      GBuffer.CustomData     = D
      return GBuffer
```

> [!note]- UniformBuffer 経由でのテクスチャバインド
> `FSceneTextureUniformParameters` が `GBufferATexture` 等をまとめて保持し、
> `IMPLEMENT_GLOBAL_SHADER_PARAMETER_STRUCT` で `"SceneTextures"` という
> HLSL バインディング名として公開される。
> シェーダーは `SceneTextures.GBufferATexture` のように参照する。
