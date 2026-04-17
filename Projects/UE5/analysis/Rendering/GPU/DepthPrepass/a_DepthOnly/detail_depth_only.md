# Depth Only シェーダー詳細

- グループ: a - DepthOnly
- GPU 概要: [[01_depthprepass_gpu_overview]]
- CPU 詳細: [[a_depth_prepass]]
- リファレンス: [[ref_depth_only]]

---

## 概要

**Depth Pre-pass** の GPU 実装。Opaque メッシュは位置専用ストリームで高速に深度のみ書き込む。  
Masked メッシュはフルストリームで PS を呼び出し Opacity テストを行う。  
PDO（Pixel Depth Offset）マテリアルも PS で出力深度をオフセットできる。

---

## レンダリングパスの構成

```
Depth Pre-pass
  │
  ├─ [Opaque / Solid ブレンド]
  │   VS: DepthOnlyVertexShader.usf#Main()
  │   PS: なし（深度のみ書き込み）
  │   位置専用ストリーム（EVertexInputStreamType::PositionOnly）
  │   → WorldPos → WPO 適用 → ClipPos → SV_Position のみ出力
  │
  └─ [Masked / PDO]
      VS: DepthOnlyVertexShader.usf#Main()
      PS: DepthOnlyPixelShader.usf#Main()
      フルストリーム（UV, Normal, Tangent 等）
      → Opacity テスト（clip()）
      → OUTPUT_PIXEL_DEPTH_OFFSET 時は深度値を調整して出力
```

---

## 入出力

### 入力（Vertex Shader）

| リソース | 説明 |
|---------|------|
| `FVertexFactoryInput` | メッシュのジオメトリ入力（VertexFactory 別）|
| `Material.ush`（Generated）| WPO マテリアルコード |

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| `SV_POSITION` | `float4` | クリップ空間座標（深度バッファに書き込み）|
| `SV_Target0`（Masked のみ）| `float4` | OutColor（通常 0 埋め、深度のみ有効）|
| `SV_DepthGreaterEqual`（PDO 時）| `float` | オフセット後深度 |

---

## シェーダーコアロジック

### VS: DepthOnlyVertexShader.usf#Main()

```hlsl
void Main(
    FVertexFactoryInput Input,
    out FDepthOnlyVSOutput Output
    , out FStereoVSOutput StereoOutput)
{
    FVertexFactoryIntermediates VFIntermediates = GetVertexFactoryIntermediates(Input);
    float4 WorldPos = VertexFactoryGetWorldPosition(Input, VFIntermediates);

    // WPO（World Position Offset）適用
    // MATERIALBLENDING_SOLID = 0（Masked）の時のみ補間値を出力
    {
        WorldPos.xyz += GetMaterialWorldPositionOffset(VertexParameters);
    }

    Output.Position = mul(WorldPos, View.TranslatedWorldToClip);

    // MATERIALBLENDING_SOLID == 0 時のみ FactoryInterpolants を出力（PS 用）
    #if !MATERIALBLENDING_SOLID || OUTPUT_PIXEL_DEPTH_OFFSET
        Output.FactoryInterpolants = VertexFactoryGetInterpolantsVSToPS(Input, VFIntermediates, VertexParameters);
    #endif
}
```

### PS: DepthOnlyPixelShader.usf#Main()（Masked のみ実行）

```hlsl
void Main(
    in float4 SvPosition : SV_Position,
    FVertexFactoryInterpolantsVSToPS FactoryInterpolants,
    ...
    out float4 OutColor : SV_Target0
    #if OUTPUT_PIXEL_DEPTH_OFFSET
        , out float OutDepth : SV_DepthGreaterEqual
    #endif
    )
{
    FMaterialPixelParameters MaterialParameters = GetMaterialPixelParameters(FactoryInterpolants, SvPosition);
    FPixelMaterialInputs PixelMaterialInputs;
    CalcMaterialParameters(MaterialParameters, PixelMaterialInputs, SvPosition, bIsFrontFace);

    // Opacity テスト（Masked のみ）
    GetMaterialClippingPS(MaterialParameters, PixelMaterialInputs);

    // PDO（Pixel Depth Offset）
    #if OUTPUT_PIXEL_DEPTH_OFFSET
        OutDepth = ApplyPixelDepthOffset(MaterialParameters, PixelMaterialInputs, SvPosition.w);
    #endif

    OutColor = 0;
}
```

---

## Opaque vs Masked の切り替え

| 状態 | 頂点ストリーム | PS | WPO |
|-----|-----------|----|----|
| Opaque / Solid | PositionOnly | なし | あり |
| Masked | Full | あり（Opacity clip）| あり |
| PDO あり | Full | あり（深度出力）| あり |

---

## CPU 呼び出しの流れ

```
RenderPrePassView()                              // SceneVisibility.cpp
  │
  ├─ FDepthPassMeshProcessor::AddMeshBatch()
  │   EDepthDrawingMode で Opaque/Masked を判定
  │   Opaque → bUsePositionOnlyStream = true（軽量 VS のみ）
  │   Masked → bUsePositionOnlyStream = false（フル VS + PS）
  │
  ├─ DrawDynamicMeshCommands() → DrawIndexedPrimitive
  │   Opaque: VS のみ（PS なし）→ RTColorMask = 0
  │   Masked: VS + PS（clip() で Opacity テスト）
  │
  └─ Nanite メッシュ: BuildDepthRasterBin() → DepthOnlyCS を Dispatch
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `DepthOnlyVertexShader.usf` | VS | Depth Pre-pass 頂点変換 |
| `DepthOnlyPixelShader.usf` | PS | Masked Opacity テスト・PDO |
| `VelocityShader.usf` | VS/PS | Velocity Buffer 生成（Depth Pre-pass の後に実行）|
