# Base Pass Opaque シェーダー詳細

- グループ: a - Opaque
- GPU 概要: [[01_basepass_gpu_overview]]
- CPU 詳細: [[a_basepass_gbuffer]]
- リファレンス: [[ref_opaque]]

---

## 概要

**Opaque BasePass** は通常の不透明メッシュの GBuffer 書き込みを担う。  
VS が WPO・接線空間・ライトマップ UV を計算し、PS がマテリアルグラフを評価して  
GBufferA/B/C/D/E/F に法線・BaseColor・Roughness/Metallic/Specular 等を書き込む。  
DBuffer デカール（`ApplyDBufferData()`）も PS の GBuffer 書き込み前に適用される。

---

## レンダリングパスの構成

```
BasePass Opaque
  │
  ├─ VS: BasePassVertexShader.usf#Main()
  │   ├─ WPO（World Position Offset）適用
  │   ├─ 接線空間 → ワールド空間変換行列を補間値に格納
  │   ├─ ライトマップ UV（Static Lighting）
  │   └─ 半透明: フォワードライティング（TRANSLUCENCY_PERVERTEX_FORWARD_SHADING 時）
  │
  └─ PS: BasePassPixelShader.usf（内部で FPixelShaderInOut_MainPS を呼ぶ）
      ├─ マテリアルグラフ評価（GetMaterialBaseColor, GetMaterialNormal 等）
      ├─ DBuffer 適用（ApplyDBufferData）
      ├─ SubsurfaceProfile / Anisotropy / Clearcoat 等の特殊 ShadingModel 処理
      └─ GBuffer へのエンコード・書き込み
```

---

## GBuffer レイアウト

| ターゲット | フォーマット | 内容 |
|-----------|------------|------|
| GBufferA | `R8G8B8A8` | rgb = ワールド法線（Octahedron エンコード）, a = PerObjectGBufferData |
| GBufferB | `R8G8B8A8` | r = Metallic, g = Specular, b = Roughness, a = ShadingModelID |
| GBufferC | `R8G8B8A8_SRGB` | rgb = BaseColor, a = AO |
| GBufferD | `R8G8B8A8` | マテリアル固有データ（Subsurface / Anisotropy 等）|
| GBufferE | `R8G8B8A8` | 事前計算済みシャドウ / Custom Data |
| GBufferF | `R16G16` | 接線（Clearcoat / Anisotropy 用）|
| SceneDepth | `R32F` | HW 深度バッファ（Z）|
| VelocityBuffer | `R16G16F` | モーションベクター（Velocity 有効時）|

---

## シェーダーコアロジック

### VS: BasePassVertexShader.usf#Main()

```hlsl
void Main(FVertexFactoryInput Input, out FBasePassVSOutput Output)
{
    FVertexFactoryIntermediates VFIntermediates = GetVertexFactoryIntermediates(Input);
    float4 WorldPosition = VertexFactoryGetWorldPosition(Input, VFIntermediates);

    // WPO 適用
    WorldPosition.xyz += GetMaterialWorldPositionOffset(VertexParameters);

    // クリップ座標
    Output.Position = mul(WorldPosition, View.TranslatedWorldToClip);

    // 補間値（接線・法線・UV・ライトマップ UV）
    Output.FactoryInterpolants = VertexFactoryGetInterpolantsVSToPS(Input, VFIntermediates, VertexParameters);

    // 半透明フォワードライティング（TRANSLUCENCY_PERVERTEX_FORWARD_SHADING 時）
    #if TRANSLUCENCY_PERVERTEX_FORWARD_SHADING
        Output.VertexFog = GetForwardDynamicLighting(WorldPosition.xyz, ...);
    #endif
}
```

### PS: BasePassPixelShader.usf（内部 FPixelShaderInOut_MainPS）

```hlsl
// マテリアル評価 → GBuffer 書き込みの簡略フロー
void FPixelShaderInOut_MainPS(...)
{
    // 1. マテリアルパラメータ計算
    FMaterialPixelParameters MaterialParameters = GetMaterialPixelParameters(FactoryInterpolants, SvPosition);
    CalcMaterialParameters(MaterialParameters, PixelMaterialInputs, SvPosition, bIsFrontFace);

    // 2. マテリアル属性を取得
    half3 BaseColor = GetMaterialBaseColor(PixelMaterialInputs);
    half  Metallic  = GetMaterialMetallic(PixelMaterialInputs);
    half  Roughness = saturate(GetMaterialRoughness(PixelMaterialInputs));
    half3 Normal    = GetMaterialNormal(MaterialParameters, PixelMaterialInputs);

    // 3. DBuffer デカール適用（r.DBuffer=1 時）
    #if ENABLE_DBUFFER
        FDBufferData DBufferData = GetDBufferData(ScreenUV, ShadingModelID);
        ApplyDBufferData(DBufferData, Normal, SubsurfaceColor, Roughness, BaseColor, Metallic, Specular);
    #endif

    // 4. GBuffer エンコード
    SetGBufferForShadingModel(GBuffer, MaterialParameters, ...);
    // → GBufferA/B/C/D/E/F に EncodeGBuffer() で書き込み
}
```

---

## CPU 呼び出しの流れ

```
RenderBasePass()                                // SceneRendering.cpp
  │
  ├─ RenderBasePassView()
  │   ├─ FBasePassMeshProcessor::AddMeshBatch()
  │   │   DrawingPolicy で VS/PS を選択（VertexFactory / ShadingModel 別）
  │   └─ DrawDynamicMeshCommands()
  │       VS: BasePassVertexShader.usf#Main()
  │       PS: BasePassPixelShader.usf
  │       RT: GBufferA/B/C/D/E/F + SceneDepth + VelocityBuffer
  │
  └─ [Nanite] 別パスで NaniteExportGBuffer / NaniteShadeBinning を実行
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `BasePassVertexShader.usf` | VS | WPO・接線空間・ライトマップ UV 計算 |
| `BasePassPixelShader.usf` | PS | マテリアル評価・GBuffer 書き込み |
| `BasePassVertexCommon.ush` | ヘッダ | FBasePassVSOutput / 共通補間値定義 |
| `DBufferCommon.ush` | ヘッダ | `ApplyDBufferData()` (DBuffer デカール適用）|
| `ShadingModelMaterials.ush` | ヘッダ | ShadingModel 別 GBuffer エンコード |
