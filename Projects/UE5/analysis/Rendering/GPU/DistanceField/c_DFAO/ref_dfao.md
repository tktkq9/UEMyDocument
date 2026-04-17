# GPU c Ref: DFAO シェーダーリファレンス

- シェーダー: `DistanceFieldObjectCulling.usf`, `DistanceFieldScreenGridLighting.usf`, `DistanceFieldLightingPost.usf`
- CPU 対応: [[ref_dfao]]
- 上位: [[01_distance_field_gpu_overview]]

---

## シェーダーファイル一覧

| ファイル | エントリポイント | 役割 |
|---------|---------------|------|
| `DistanceFieldObjectCulling.usf` | `CullObjectsForViewCS` | フラスタムカリング |
| `DistanceFieldObjectCulling.usf` | `BuildTileObjectListCS` | タイル×オブジェクト交差リスト構築 |
| `DistanceFieldScreenGridLighting.usf` | `ComputeDistanceFieldNormalCS` | 半解像度法線計算 |
| `DistanceFieldScreenGridLighting.usf` | `AOLevelSetCS` | Global SDF コーントレース AO |
| `DistanceFieldLightingPost.usf` | `UpsampleBentNormalAOPS/CS` | BentNormal アップスケール |

---

## CullObjectsForViewCS

**ファイル**: `DistanceFieldObjectCulling.usf`

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `RWObjectIndirectArguments` | `RWBuffer<uint>` | IndirectDraw 引数（カリング通過数） |
| `RWCulledObjectData` | `RWStructuredBuffer<float4>` | カリング通過オブジェクトデータ |
| `NumSceneObjects` | `uint` | 全オブジェクト数 |
| `AOMaxViewDistance` | `float` | 最大ビュー距離 |
| `AOObjectMaxDistance` | `float` | オブジェクト影響最大距離 |

**スレッドグループ**: `[numthreads(UPDATEOBJECTS_THREADGROUP_SIZE, 1, 1)]`

---

## BuildTileObjectListCS

**ファイル**: `DistanceFieldObjectCulling.usf`

スクリーンを `32x32` ピクセルのタイルに分割し、各タイルに影響するオブジェクトリストを構築する。

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `RWTileConeAxisAndCos` | `RWStructuredBuffer<FVector4f>` | タイルのコーン軸・コサイン |
| `RWTileConeDepthRanges` | `RWStructuredBuffer<FVector4f>` | タイルの深度範囲 |
| `RWNumCulledTilesArray` | `RWStructuredBuffer<uint>` | タイルごとのオブジェクト数 |
| `RWCulledTileDataArray` | `RWBuffer<uint>` | タイル×オブジェクトインデックスリスト |
| `TileListGroupSize` | `FIntPoint` | タイルグリッド解像度 |

---

## FAOParameters（シェーダーバインド）

`DistanceFieldAmbientOcclusion.h:83`

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `AOObjectMaxDistance` | `float` | オブジェクト SDF 最大距離 |
| `AOStepScale` | `float` | コーンステップ距離スケール |
| `AOStepExponentScale` | `float` | ステップ指数成長スケール |
| `AOMaxViewDistance` | `float` | 最大ビュー距離（fp16 限界: 65000）|
| `AOGlobalMaxOcclusionDistance` | `float` | Global SDF コーントレース最大距離 |

---

## FDFAOUpsampleParameters（アップスケール）

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `BentNormalAOTexture` | `Texture2D` | 半解像度 BentNormal AO テクスチャ |
| `BentNormalAOSampler` | `SamplerState` | バイリニアサンプラー |
| `AOBufferBilinearUVMax` | `FVector2f` | UV クランプ範囲 |
| `DistanceFadeScale` | `float` | 遠距離フェードスケール |
| `AOMaxViewDistance` | `float` | フェード開始距離 |

---

## FAOScreenGridParameters（コーン可視性バッファ）

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `RWScreenGridConeVisibility` | `RWBuffer<uint>` | コーン可視性書き込みバッファ |
| `ScreenGridConeVisibility` | `Buffer<uint>` | コーン可視性読み取りバッファ |
| `ScreenGridConeVisibilitySize` | `FIntPoint` | バッファサイズ（タイル数 × コーン数） |

---

## FTileIntersectionParameters（タイル交差リスト）

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `TileConeAxisAndCos` | `StructuredBuffer<FVector4f>` | タイルのコーン方向（読み取り用）|
| `TileConeDepthRanges` | `StructuredBuffer<FVector4f>` | タイルの深度範囲（読み取り用）|
| `NumCulledTilesArray` | `StructuredBuffer<uint>` | タイルごとのオブジェクト数 |
| `CulledTileDataArray` | `Buffer<uint>` | タイル×オブジェクトインデックス |
| `TileListGroupSize` | `FIntPoint` | タイルグリッド解像度 |

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `DistanceFieldLightingShared.ush` | `LoadDFObjectData`, `LoadDFObjectBounds`, AtlasサンプリングAPI |
| `DistanceFieldAOShared.ush` | `GetSpacedVectors`, `EncodeDownsampledGBuffer`, `DecodeNormal` |
| `DistanceField/GlobalDistanceFieldShared.ush` | `SampleGlobalDistanceField`, `GetClipmapUVFromWorldPos` |

---

## 定数

| 定数名 | 値 | 説明 |
|--------|-----|------|
| `GAODownsampleFactor` | `2` | AO の処理解像度スケールダウン係数 |
| `CulledTileDataStride` | `2` | タイルデータ1エントリのストライド |
| `ConeTraceObjectsThreadGroupSize` | `64` | コーントレース CS スレッドグループサイズ |
| `DOWNSAMPLE_FACTOR` | `2` | シェーダー内マクロ（GAODownsampleFactor と同値）|

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.DistanceFieldAO` | DFAO 有効/無効 |
| `r.AOQuality` | 品質レベル（コーン数・ステップ数切り替え） |
| `r.AOGlobalDFStartDistance` | Global SDF トレース開始距離 |
| `r.AOMaxViewDistance` | AO 最大ビュー距離 |
| `r.DistanceFieldAO.ApplyToStaticIndirect` | 静的間接ライティングへの適用 |
