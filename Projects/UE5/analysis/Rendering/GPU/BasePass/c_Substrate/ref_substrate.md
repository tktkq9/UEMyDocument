# REF: Substrate GBuffer エンコード シェーダー

- グループ: c - Substrate
- 詳細: [[detail_substrate]]
- CPU リファレンス: [[ref_basepass_gbuffer]]
- ソース: `Engine/Shaders/Private/Substrate/Substrate.ush`  
          `Engine/Shaders/Private/Substrate/SubstrateExport.ush`

---

## Substrate.ush 主要関数

### GetSubstrateData（マテリアル評価）

```hlsl
FSubstrateData GetSubstrateData(
    FMaterialPixelParameters MaterialParameters,
    FPixelMaterialInputs PixelMaterialInputs,
    inout float3 EmissiveColor,
    inout float Opacity)
// マテリアルグラフの Substrate ノードを評価して FSubstrateData を返す
// → FSubstrateData.Slabs[] に各 BSDF レイヤーが格納される
```

### FSubstrateSlab（BSDF レイヤー構造体）

```hlsl
struct FSubstrateSlab
{
    float3 DiffuseAlbedo;   // ディフューズアルベド
    float3 F0;              // フレネル 0 度反射率（メタリック → BaseColor に収束）
    float  Roughness;       // 表面粗さ
    float3 Normal;          // 法線（ワールド空間）
    float  Anisotropy;      // 異方性
    float3 Tangent;         // 接線（異方性用）
    float  SSS_MFP;         // Subsurface Mean Free Path
    float3 Emissive;        // 自発光
    float  Coverage;        // レイヤーカバレッジ（0〜1）
};
```

### SubstrateConvertToLegacyGBuffer（レガシー変換）

```hlsl
void SubstrateConvertToLegacyGBuffer(
    FSubstrateData SubstrateData,
    out FGBufferData OutGBufferData)
// Substrate BSDF を従来 GBuffer フォーマットに近似変換
// → r.Substrate.GBufferFormat=0 時（後方互換）
```

---

## SubstrateExport.ush 主要関数

### SubstratePixelExport

```hlsl
// BasePassPixelShader.usf から呼ばれる
// GBuffer / SubstrateBuffer への最終書き込み
void SubstratePixelExport(
    inout FPixelShaderOut Out,
    FSubstrateData SubstrateData,
    ...)
{
    #if SUBSTRATE_GBUFFER_FORMAT == 1
        // SubstrateBuffer に直接書き込み（UAV）
        WriteSubstrateBuffer(SubstrateData, Out);
    #else
        // 従来 GBuffer に変換して書き込み
        SubstrateConvertToLegacyGBuffer(SubstrateData, LegacyGBuffer);
        EncodeGBuffer(LegacyGBuffer, Out.MRT);
    #endif
}
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Substrate` | 0 | Substrate マテリアルシステム有効 |
| `r.Substrate.GBufferFormat` | 0 | 0=レガシー互換 / 1=Substrate 専用フォーマット |
| `r.Substrate.BytesPerPixel` | 80 | SubstrateBuffer の 1 ピクセルあたりのバイト数 |
| `r.Substrate.ClassificationAsync` | 1 | タイル分類を AsyncCompute で実行 |

---

> [!note]- Substrate と従来 ShadingModel の違い
> 従来の GBuffer は ShadingModelID（4bit）で固定の計算を切り替えていた。  
> Substrate は任意の BSDF レイヤーを組み合わせられるため、より柔軟なマテリアル表現が可能。  
> ただし Deferred Lighting パスが複雑になるため、Substrate タイル分類で「シンプルな領域」と  
> 「複雑な領域」を分離し、シンプルな領域には軽量パスを使う最適化が入っている。
