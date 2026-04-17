# Light Functions シェーダー詳細

- グループ: c - LightFunctions
- GPU 概要: [[01_deferred_lighting_gpu_overview]]
- CPU 詳細: [[c_light_functions]]
- リファレンス: [[ref_light_functions]]

---

## 概要

**Light Function** はライトに適用するマテリアルテクスチャで、ライトの形状や色を変調する。  
UE5.3 以降は **Light Function Atlas** を使い、全ライト関数を 1 枚のアトラステクスチャに事前レンダリングして  
ライティング計算時に参照する（従来の個別パスより効率的）。

---

## レンダリングパスの構成

```
Light Function Atlas レンダリング
  │
  ├─ [各 Light Function マテリアルを Atlas タイルに焼き込む]
  │   VS: LightFunctionVertexShader.usf（フルスクリーンクワッド）
  │   PS: LightFunctionPixelShader.usf
  │   マテリアルグラフを評価 → Atlas テクスチャの各タイルに書き込み
  │
  └─ [Deferred Lighting 時に Atlas からサンプル]
      LightData.LightFunctionAtlasIndex → Atlas.SampleLevel(UV, Mip)
      → ライト色に乗算

[従来方式（Light Function Atlas なし）]
  ├─ 各ライトに個別パスを実行
  └─ LightAttenuation テクスチャに Light Function 結果を書き込む
```

---

## シェーダーコアロジック

### LightFunctionPixelShader.usf

```hlsl
// ライト空間の UV からマテリアルを評価して Light Function 値を返す
void Main(
    float2 TexCoord : TEXCOORD0,
    float4 SvPosition : SV_POSITION,
    out float4 OutColor : SV_Target0)
{
    // ライト空間 UV からワールド座標を復元
    float3 LightSpacePos = GetLightSpacePosition(TexCoord);

    // マテリアルグラフ評価
    FMaterialPixelParameters MaterialParameters = GetMaterialPixelParameters_LightFunction(LightSpacePos);
    FPixelMaterialInputs PixelMaterialInputs;
    CalcMaterialParameters(MaterialParameters, PixelMaterialInputs, ...);

    // Emissive チャンネルを Light Function 値として使用
    float3 LightFunction = GetMaterialEmissive(PixelMaterialInputs);

    OutColor = float4(LightFunction, 1);
}
```

### Deferred Lighting での Light Function 適用

```hlsl
// DeferredLightingCommon.ush
float3 LightFunctionColor = LightFunctionAtlas.SampleLevel(
    Sampler,
    GetLightFunctionAtlasUV(LightData, WorldPosition),
    0).rgb;

// ライト色に乗算
float3 LightColor = LightData.Color * LightFunctionColor;
```

---

## CPU 呼び出しの流れ

```
RenderLightFunctionAtlas()                   // LightFunctionAtlasRendering.cpp
  │
  ├─ 各 Light Function マテリアルをアトラスタイルに割り当て
  ├─ AddPass(RDG_EVENT_NAME("LightFunctionAtlas"))
  │   PS: LightFunctionPixelShader.usf
  │   RT: LightFunctionAtlasTexture（各タイルに書き込み）
  │
  └─ LightData.LightFunctionAtlasIndex にタイルインデックスを格納
      → Deferred Lighting パスで参照
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `LightFunctionPixelShader.usf` | PS | Light Function マテリアル評価 |
| `LightFunctionVertexShader.usf` | VS | フルスクリーンクワッド（Atlas タイル範囲に制限）|
| `LightFunctionAtlas/LightFunctionAtlasRender.usf` | PS/CS | Atlas タイルへのレンダリング |
| `LightFunctionAtlas/LightFunctionAtlasCommon.ush` | ヘッダ | Atlas UV 計算・バインドユーティリティ |
