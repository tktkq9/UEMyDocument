# リファレンス：FSubstrateSceneData / FSubstrateViewData（Substrate.h）

- グループ: a - マテリアルデータ構造
- 上位: [[a_substrate_material]]
- 関連: [[ref_substrate_params]] | [[ref_substrate_classify]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Substrate/Substrate.h`

## 概要

Substrate システムのシーン単位・ビュー単位のデータを保持する主要構造体。  
`FScene` が `FSubstrateSceneData` を、`FViewInfo` が `FSubstrateViewData` を保持する。

---

## `FSubstrateSceneData`

### メンバ変数（容量追跡）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `ViewsMaxBytesPerPixel` | `uint32` | このフレームの全ビューの最大 BytesPerPixel |
| `ViewsMaxClosurePerPixel` | `uint32` | このフレームの全ビューの最大 ClosurePerPixel |
| `PersistentMaxBytesPerPixel` | `uint32` | シーン作成以来の累積最大 BytesPerPixel |
| `PersistentMaxClosurePerPixel` | `uint32` | シーン作成以来の累積最大 ClosurePerPixel |
| `EffectiveMaxBytesPerPixel` | `uint32` | 実際に使用中の最大バイト数 |
| `EffectiveMaxClosurePerPixel` | `uint32` | 実際に使用中の最大クロージャ数 |
| `UsesTileTypeMask` | `uint8` | 使用中のタイル種別ビットマスク |

### メンバ変数（機能フラグ）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `bUsesAnisotropy` | `bool` | 異方性マテリアルを含むか |
| `bRoughDiffuse` | `bool` | Rough Diffuse モデル有効 |
| `bRoughnessTracking` | `bool` | ラフネストラッキング有効（上位レイヤーのラフネスが下位に伝播） |
| `bStochasticLighting` | `bool` | Stochastic Lighting 有効 |
| `PeelLayersAboveDepth` | `int32` | デバッグ用レイヤーピール深度（-1=無効） |

### メンバ変数（デバッグスライス）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `SliceStoringDebugSubstrateTreeDataWithoutMRT` | `int32` | MRT なしデバッグデータのスライスインデックス |
| `SliceStoringDebugSubstrateTreeData` | `int32` | デバッグデータのスライスインデックス |
| `FirstSliceStoringSubstrateSSSDataWithoutMRT` | `int32` | MRT なし SSS データの先頭スライス |
| `FirstSliceStoringSubstrateSSSData` | `int32` | SSS データの先頭スライス |

### メンバ変数（フレームリソース）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `MaterialTextureArray` | `FRDGTextureRef` | クロージャデータ（Texture2DArray&lt;uint&gt;） |
| `MaterialTextureArrayUAVWithoutRTs` | `FRDGTextureUAVRef` | UAV（MRT なしBasePass 用） |
| `MaterialTextureArrayUAV` | `FRDGTextureUAVRef` | UAV（MRT ありBasePass 用） |
| `MaterialTextureArraySRV` | `FRDGTextureSRVRef` | SRV（読み取り用） |
| `TopLayerTexture` | `FRDGTextureRef` | 最上位レイヤー情報 |
| `OpaqueRoughRefractionTexture` | `FRDGTextureRef` | 粗い屈折テクスチャ |
| `ClosureOffsetTexture` | `FRDGTextureRef` | クロージャオフセット |
| `SampledMaterialTexture` | `FRDGTextureRef` | サンプル済みマテリアル |
| `SeparatedSubSurfaceSceneColor` | `FRDGTextureRef` | SSS 分離バッファ |
| `SeparatedOpaqueRoughRefractionSceneColor` | `FRDGTextureRef` | 屈折分離バッファ |
| `SubstratePublicGlobalUniformParameters` | `TRDGUniformBufferRef<...>` | 公開グローバル UBO |

---

## `FSubstrateViewData`

### メンバ変数（容量）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `MaxClosurePerPixel` | `uint32` | このビューの最大クロージャ数 |
| `MaxBytesPerPixel` | `uint32` | このビューの最大バイト数 |
| `UsesTileTypeMask` | `uint8` | 使用中タイル種別のビットマスク |
| `bUsesAnisotropy` | `bool` | 異方性マテリアルを含むか |
| `TileCount` | `FIntPoint` | タイルグリッドのサイズ |
| `TileEncoding` | `uint32` | タイル座標エンコーディング（16bit / 8bit） |
| `LayerCount` | `uint32` | レイヤー数 |

### メンバ変数（タイルリストバッファ）

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `ClassificationTileListBuffer` | `FRDGBufferRef` | タイル座標リスト（全種別） |
| `ClassificationTileListBufferSRV` | `FRDGBufferSRVRef` | SRV |
| `ClassificationTileListBufferUAV` | `FRDGBufferUAVRef` | UAV（書き込み時） |
| `ClassificationTileListBufferOffset[SUBSTRATE_TILE_TYPE_COUNT]` | `uint32[]` | 種別ごとの開始オフセット |
| `ClassificationTileDrawIndirectBuffer` | `FRDGBufferRef` | Indirect Draw 引数 |
| `ClassificationTileDispatchIndirectBuffer` | `FRDGBufferRef` | Indirect Dispatch 引数 |
| `ClosureTileBuffer` | `FRDGBufferRef` | クロージャ単位のタイルバッファ |
| `ClosureTileCountBuffer` | `FRDGBufferRef` | クロージャタイル数バッファ |
| `ClosureTileDispatchIndirectBuffer` | `FRDGBufferRef` | クロージャタイル Indirect Dispatch 引数 |
| `ClosureTileRaytracingIndirectBuffer` | `FRDGBufferRef` | RT 用 Indirect Dispatch 引数 |

### 主要メソッド

| メソッド | 説明 |
|---------|------|
| `Reset()` | フレーム開始時の状態リセット |

---

## `FSubstrateViewDebugData`

デバッグビジュアライゼーション用のビュー別データ：

| メンバ | 説明 |
|--------|------|
| `PixelMaterialDebugDataSizeBytes` | ピクセルマテリアルデバッグバッファサイズ |
| `PixelMaterialDebugDataReadbackQueries` | GPU リードバックキュー |
| `CreateTransientPixelDebugBuffer()` | 一時デバッグバッファ生成 |

---

## ユーティリティ関数

| 関数 | 説明 |
|------|------|
| `Substrate::GetSubstrateTextureResolution(View, InResolution)` | Substrate テクスチャの実解像度を返す |
| `Substrate::GetSubstrateMaxClosureCount(View)` | ビューの最大クロージャ数を返す |
| `Substrate::GetSubstrateUsesAnisotropy(View)` | 異方性マテリアルを含むか |
| `Substrate::UsesSubstrateMaterialBuffer(Platform)` | プラットフォームが Substrate バッファを使うか |
| `Substrate::GetSubstrateUsesTileType(View, TileType)` | 特定タイル種別が使用中か |

---

## 使用箇所

- `Substrate::InitialiseSubstrateFrameSceneData()` — `FSubstrateSceneData` の全リソースを RDG 上に初期化
- `FViewInfo` — `FSubstrateViewData` を内包
- `FScene` — `FSubstrateSceneData` を内包
- [[ref_substrate_params]] — `FSubstrateGlobalUniformParameters` にバインドして GPU に転送
