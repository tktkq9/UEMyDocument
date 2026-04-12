# REF: Nanite Shading シェーダー

- グループ: c - Shading
- 詳細: [[detail_shading]]
- CPU リファレンス: [[ref_nanite_shading]] | [[ref_nanite_draw_list]]
- ソース: `Engine/Shaders/Private/Nanite/NaniteShadeBinning.usf`
          `Engine/Shaders/Private/Nanite/NaniteExportGBuffer.usf`

---

## NaniteShadeBinning.usf

### ShadingBinBuildCS

```hlsl
[numthreads(SHADING_BIN_TILE_THREADS, 1, 1)]
void ShadingBinBuildCS(
    uint ThreadIndex : SV_GroupIndex,
    uint2 GroupId    : SV_GroupID
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | VisBuffer の全ピクセルをマテリアルシェーディングビンに分類。Quad 単位でフルタイル / ルーズに仕分け |
| **入力** | `VisBuffer64`, `RasterBinMeta`, `ShadingBinMetaData` |
| **出力** | `OutShadingBinData`（ピクセル座標リスト）, `OutShadingBinMeta`（カウント）|
| **CPU 関数** | `Nanite::BuildShadingCommands()` |
| **注意** | `SHADING_BIN_TILE_SIZE` × `SHADING_BIN_TILE_SIZE` のタイル単位で処理。Z-Order カーブでキャッシュ効率向上 |

### ShadingBinReserveCS

```hlsl
[numthreads(64, 1, 1)]
void ShadingBinReserveCS(uint ShadingBin : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 各ビンのピクセル数から Indirect Dispatch 引数（ThreadGroupCountX）を計算して書き出し |
| **入力** | `ShadingBinCount`, `ShadingBinMetaData`（ElementCount）|
| **出力** | `OutShadingBinArgs`（uint4: ThreadGroupCountXYZ + reserved）|
| **CPU 関数** | `Nanite::BuildShadingCommands()` |

### ShadingBinValidateCS

```hlsl
[numthreads(64, 1, 1)]
void ShadingBinValidateCS(uint ShadingBin : SV_DispatchThreadID)
```

| 目的 | デバッグ専用 — ビンに実際に書き込まれたカウントが期待値と一致するか検証（`PLATFORM_BREAK`）|

### ClearCMaskRectCS

```hlsl
[numthreads(8, 8, 1)]
void ClearCMaskRectCS(uint2 DispatchThreadID : SV_DispatchThreadID)
```

| 目的 | RenderTarget のタイル単位 CMask（色圧縮メタデータ）を指定矩形でクリア |

---

## NaniteExportGBuffer.usf

### MarkStencilPS

```hlsl
void MarkStencilPS(
    in float4 SvPosition : SV_Position,
    out float OutDepth   : SV_Depth
)
```

| 目的 | Nanite ピクセルが存在する位置にステンシルバッファをマーク（非 Nanite ピクセルは `discard`）|

### EmitSceneDepthPS

```hlsl
void EmitSceneDepthPS(
    in float4  SvPosition         : SV_Position,
    out uint   OutShadingMask     : SV_Target0,  // SHADING_MASK_EXPORT 時
    out float4 OutVelocity        : SV_Target1,  // VELOCITY_EXPORT 時
    out float  OutDepth           : SV_Depth
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | VisBuffer64 の深度を SceneDepth に書き出す。同時に ShadingMask と Velocity も出力 |
| **入力** | `VisBuffer64`, `RasterBinMeta`, `InstanceSceneData`, `ViewRect` |
| **出力** | `SV_Depth`（SceneDepth）, `OutShadingMask`（ShadingBin + Nanite フラグ）, `OutVelocity` |
| **CPU 関数** | `Nanite::DispatchBasePass()` の第1フェーズ |

### EmitSceneStencilPS

```hlsl
void EmitSceneStencilPS(
    in float4 SvPosition : SV_Position,
    out float OutDepth   : SV_Depth
)
```

| 目的 | `ShadingMask` を読み取り、デカール受信ピクセルに `StencilDecal` 値でステンシルを書き込む |
| 条件 | `SHADING_MASK_LOAD=1` |

### EmitCustomDepthStencilPS

```hlsl
void EmitCustomDepthStencilPS(
    in float4 SvPosition  : SV_Position,
    out uint2 OutStencil  : SV_Target0,   // WRITE_CUSTOM_STENCIL 時
    out float OutDepth    : SV_Depth
)
```

| 目的 | Custom Depth/Stencil テクスチャへ Nanite ピクセルの深度・ステンシル値を書き込む |

### EmitShadowMapPS

```hlsl
void EmitShadowMapPS(
    in float4 SvPosition : SV_Position,
    out float OutDepth   : SV_Depth
)
```

| 目的 | Nanite ピクセルの深度を Shadow Map RT に出力（`NaniteEmitShadow.usf` と連携）|

### EmitHitProxyIdPS

```hlsl
void EmitHitProxyIdPS(
    in float4 SvPosition          : SV_Position,
    out float4 OutHitProxyId      : SV_Target0,
    out float  OutDepth           : SV_Depth
)
```

| 目的 | エディタの Hit Proxy カラーバッファに Nanite プリミティブ ID を書き込む（エディタ選択用）|

### EmitEditorSelectionDepthPS

```hlsl
void EmitEditorSelectionDepthPS(
    in float4 SvPosition : SV_Position,
    out float OutDepth   : SV_Depth
)
```

| 目的 | エディタで選択中のオブジェクトのアウトライン描画用に深度を書き込む |

---

## 主要シェーダーパラメーター（NaniteExportGBuffer.usf）

| パラメーター | 型 | 内容 |
|------------|-----|------|
| `VisBuffer64` | `Texture2D<UlongType>` | Nanite Visibility Buffer（depth + clusterID + triIdx）|
| `ShadingMask` | `Texture2D<uint>` | ピクセル分類マスク（SHADING_MASK_LOAD 時）|
| `RasterBinMeta` | `StructuredBuffer<FNaniteRasterBinMeta>` | マテリアルビンメタデータ |
| `RegularMaterialRasterBinCount` | `uint` | 通常マテリアルのビン数（オフセット計算用）|
| `MeshPassIndex` | `uint` | パスインデックス（シェーディングビン参照用）|
| `ViewRect` | `int4` | 有効ピクセル範囲 |
| `StencilClear` | `uint` | ステンシルクリア値 |
| `StencilDecal` | `uint` | デカール受信ステンシル値 |

---

> [!note]- Shade Binning の Z-Order カーブとキャッシュ効率
> `ShadingBinBuildCS` では `ZOrder2D(ThreadIndex, 3)` で 8×8 タイル内のスレッドを Z-Order（Morton Order）にマップする。  
> これにより隣接ピクセルを同じ Wave 内で処理する確率が上がり、VisBuffer64 の L1/L2 キャッシュヒット率が向上する。  
> テクスチャの読み取りパターンがランダムになりやすい VisBuffer ピクセル処理において特に効果的。

> [!note]- スカラライズ（ScalarizeForEachMatching）による Wave 最適化
> 同一 Wave 内でも異なるマテリアルビンのピクセルが混在する。`BinShadingQuad` 内の `ScalarizeForEachMatching` は  
> Wave 内で同じ ShadingBin 番号を持つスレッドを Wave アクティブマスクで集約し、  
> 1 Wave で複数ビンのカウントを効率的に処理する（アトミック操作回数を削減）。

> [!note]- ShadingMask の役割
> `EmitSceneDepthPS` が書き込む `ShadingMask` は、後続の `EmitSceneStencilPS` と BasePass マテリアルシェーダーが参照する。  
> このマスクによって「どのピクセルが Nanite ピクセルか」「そのピクセルのマテリアルビン番号」「デカール受信フラグ」が伝播し、  
> 不必要なマテリアル評価や Deferred Decal の計算をスキップできる。
