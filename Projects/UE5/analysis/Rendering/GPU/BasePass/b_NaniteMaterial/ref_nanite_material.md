# REF: Nanite Material GBuffer シェーダー

- グループ: b - NaniteMaterial
- 詳細: [[detail_nanite_material]]
- CPU リファレンス: [[ref_nanite_shading]]
- ソース: `Engine/Shaders/Private/Nanite/NaniteShadeBinning.usf`  
          `Engine/Shaders/Private/Nanite/NaniteExportGBuffer.usf`

---

## ShadeBinBuildCS（NaniteShadeBinning.usf:886）

```hlsl
[numthreads(SHADING_BIN_TILE_THREADS, 1, 1)]
void ShadeBinBuildCS(uint ThreadIndex : SV_GroupIndex, uint2 GroupId : SV_GroupID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | VisibilityBuffer を走査して各 ShadingBin のピクセル数をカウント |
| **出力** | `ShadingBinPixelCount[]` バッファ |

---

## ShadeBinReserveCS（NaniteShadeBinning.usf:903）

```hlsl
[numthreads(64, 1, 1)]
void ShadeBinReserveCS(uint ShadingBin : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Prefix Sum で各 ShadingBin の開始オフセットを計算 |
| **出力** | `ShadingBinOffset[]`（各 Bin のピクセルリスト書き込み先）|

---

## EmitSceneDepthPS（NaniteExportGBuffer.usf:70）

```hlsl
void EmitSceneDepthPS(
    in float4 SvPosition : SV_POSITION,
    out float OutDepth : SV_Depth)
```

| 項目 | 内容 |
|-----|------|
| **目的** | VisibilityBuffer の Depth 成分（上位 32bit）を SceneDepth RT に書き込む |
| **実行条件** | Nanite メッシュがある場合のみ |

---

## EmitSceneStencilPS（NaniteExportGBuffer.usf:159）

```hlsl
void EmitSceneStencilPS(
    in float4 SvPosition : SV_POSITION,
    out uint OutStencil : SV_StencilRef)
```

| 項目 | 内容 |
|-----|------|
| **目的** | VisibilityBuffer → Stencil（ステンシルバッファ）に変換 |
| **出力** | デカール・光源境界等に使う Stencil 値 |

---

## MarkStencilPS（NaniteExportGBuffer.usf:46）

```hlsl
void MarkStencilPS(
    in float4 SvPosition : SV_POSITION,
    out uint OutStencil : SV_StencilRef)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Nanite ピクセルに専用 Stencil マークを付ける（通常メッシュと区別するため）|

---

## VisibilityBuffer フォーマット

```hlsl
// VisBuffer64: uint2（64bit）
// 上位 32bit: asuint(DeviceZ)    … Reverse-Z 深度値
// 下位 32bit: VisibleClusterIndex（ClusterID を圧縮）
//             bits 0〜6  = TriangleIndex（クラスター内）
//             bits 7〜31 = VisibleClusterIndex（全シーン）

// デコード例:
uint2 VisData = InVisBuffer64[PixelPos];
float DeviceZ = asfloat(VisData.y);
uint VisibleClusterIndex = VisData.x >> 7;
uint TriangleIndex = VisData.x & 0x7F;
```

---

> [!note]- Nanite ShadeBinning と通常 BasePass の違い
> 通常の BasePass は「メッシュ単位」でシェーダーを切り替えながら描画するが、  
> Nanite は全メッシュを一括ラスタライズして VisibilityBuffer に書き込み、  
> その後「マテリアル種類（ShadingBin）単位」でまとめて Compute Shading を行う。  
> これにより Draw Call 数が劇的に削減され、Triangle Throughput ボトルネックを回避できる。
