# GPU b Ref: Global SDF シェーダーリファレンス

- シェーダー: `DistanceField/GlobalDistanceField*.usf`
- CPU 対応: [[ref_global_sdf]]
- 上位: [[01_distance_field_gpu_overview]]

---

## シェーダーファイル一覧

| ファイル | 主要エントリ | 役割 |
|---------|-----------|------|
| `GlobalDistanceField.usf` | `InitializePageFreeListCS`<br>`AllocatePagesCS`<br>`PageClearCS` | ページ管理・初期化・クリア |
| `GlobalDistanceFieldCompositeObjects.usf` | `ComposeObjectsIntoPagesCS` | Mesh SDF → Global SDF ページ合成 |
| `GlobalDistanceFieldMip.usf` | `GenerateMipCS` | 粗い解像度 Mip テクスチャ生成 |
| `GlobalDistanceFieldHeightfields.usf` | `ComposeHeightfieldsIntoClipmap` | ランドスケープ SDF 合成 |
| `GlobalDistanceFieldDebug.usf` | `VisualizeSDF*` | デバッグ可視化 |

---

## 主要 CS エントリポイント

### InitializePageFreeListCS（GlobalDistanceField.usf）

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `RWPageFreeListAllocator` | `RWStructuredBuffer<uint>` | フリーリストアロケータ |
| `RWPageFreeList` | `RWBuffer<uint>` | フリーページリスト |
| `MaxPageNum` | `uint` | 最大ページ数 |

---

### AllocatePagesCS（GlobalDistanceField.usf）

更新が必要なクリップマップ領域に対して PageAtlas のページを割り当てる。

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `RWPageTableCombinedTexture` | `RWTexture3D<uint>` | 出力: ページテーブル |
| `RWPageFreeListAllocator` | `RWStructuredBuffer<uint>` | ページ割り当てカウンタ |
| `UpdateBoundsBuffer` | `StructuredBuffer<FUpdateBounds>` | 更新領域リスト |

---

### ComposeObjectsIntoPagesCS（GlobalDistanceFieldCompositeObjects.usf）

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `RWPageAtlasTexture` | `RWTexture3D<UNORM float>` | SDF 値アトラス |
| `RWCoverageAtlasTexture` | `RWTexture3D<UNORM float>` | カバレッジアトラス |
| `PageTableLayerTexture` | `Texture3D<uint>` | ページテーブル（入力）|
| `CullGridObjectHeader` | `StructuredBuffer<uint>` | CullGrid ヘッダ |
| `CullGridObjectArray` | `StructuredBuffer<uint>` | CullGrid オブジェクトリスト |
| `ComposeTileBuffer` | `StructuredBuffer<uint>` | 合成タイルリスト |
| `InfluenceRadius` | `float` | 合成影響半径 |
| `InfluenceRadiusSq` | `float` | 影響半径の二乗 |
| `ClipmapResolution` | `uint3` | クリップマップ解像度 |
| `ClipmapVoxelExtent` | `float` | ボクセル1辺の実距離 |
| `ClipmapMinBounds` | `float3` | クリップマップの最小境界 |
| `PreViewTranslationHigh/Low` | `float3` | 高精度プリビュー変換（FDFVector3 用）|

**スレッドグループ**: `[numthreads(4, 4, 4)]`（1ページ内のボクセル単位）

---

### GenerateMipCS（GlobalDistanceFieldMip.usf）

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `PageAtlasTexture` | `Texture3D<float>` | 入力: SDF ページアトラス |
| `RWMipTexture` | `RWTexture3D<float>` | 出力: Mip テクスチャ |
| `MipClipmapResolution` | `uint3` | 出力 Mip の解像度 |
| `MipFactor` | `float` | ページ → Mip のスケール係数 |

---

## FGlobalDistanceFieldParameters2（シェーダーバインド）

`GlobalDistanceFieldParameters.h:46` → `GlobalDistanceField/GlobalDistanceFieldShared.ush`

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `GlobalDistanceFieldPageAtlasTexture` | `Texture3D` | SDF 値アトラス |
| `GlobalDistanceFieldCoverageAtlasTexture` | `Texture3D` | カバレッジアトラス |
| `GlobalDistanceFieldPageTableTexture` | `Texture3D<uint>` | ページテーブル |
| `GlobalDistanceFieldMipTexture` | `Texture3D` | 粗い Mip テクスチャ |
| `GlobalVolumeTranslatedCenterAndExtent[MaxClipmaps]` | `FVector4f[]` | 各クリップマップの中心・範囲 |
| `GlobalVolumeTranslatedWorldToUVAddAndMul[MaxClipmaps]` | `FVector4f[]` | ワールド→UV 変換 |
| `GlobalDistanceFieldMipFactor` | `float` | Mip スケール係数 |
| `GlobalDistanceFieldMipTransition` | `float` | Mip ブレンド遷移 |
| `GlobalDistanceFieldClipmapSizeInPages` | `int32` | クリップマップのページ数 |
| `GlobalDistanceFieldInvPageAtlasSize` | `FVector3f` | 1 / PageAtlasSize |
| `GlobalVolumeDimension` | `float` | ボクセル解像度 |
| `GlobalVolumeTexelSize` | `float` | 1 / VolumeDimension |
| `MaxGlobalDFAOConeDistance` | `float` | DFAO コーントレース最大距離 |
| `NumGlobalSDFClipmaps` | `uint` | 有効クリップマップ段数 |

---

## GlobalDistanceFieldShared.ush 主要関数

| 関数 | 説明 |
|------|------|
| `GetGlobalDistanceFieldPage(PageTable, UVW, Clipmap)` | ページテーブルからページ番号を取得 |
| `SampleGlobalDistanceField(PageAtlas, PageTable, UV)` | クリップマップをサンプリング |
| `GetClipmapUVFromWorldPos(TranslatedWorldPos, ClipmapIndex)` | ワールド座標 → クリップマップ UV |
| `GetGlobalDistanceFieldMipUV(TranslatedWorldPos, ClipmapIndex)` | Mip テクスチャの UV を計算 |

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.AOGlobalDistanceField` | Global SDF 有効フラグ |
| `r.GlobalDistanceField.MaxPageNum` | 最大ページ数 |
| `r.GlobalDistanceField.ClipmapExtent` | クリップマップ範囲係数 |
| `r.GlobalDistanceField.Resolution` | 解像度オーバーライド |
| `r.GlobalDistanceField.UpdateFrequency` | 更新頻度制限 |
