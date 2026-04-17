# GPU a Ref: Mesh SDF シェーダーリファレンス

- シェーダー: `DistanceFieldDownsampling.usf`, `DistanceFieldStreaming.usf`
- CPU 対応: [[ref_df_atlas]]
- 上位: [[01_distance_field_gpu_overview]]

---

## シェーダーファイル一覧

| ファイル | エントリポイント | 役割 |
|---------|---------------|------|
| `DistanceFieldStreaming.usf` | `ScatterUploadCS` | Mesh SDF ボクセルを Atlas に書き込み |
| `DistanceFieldDownsampling.usf` | `DownsampleMeshSDFCS` | Mip ダウンサンプリング |

---

## ScatterUploadCS

**ファイル**: `DistanceFieldStreaming.usf`

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `RWAtlasTexture` | `RWTexture3D<float>` | 書き込み先 SDF Atlas |
| `UploadBuffer` | `Buffer<uint4>` | {DestX, DestY, DestZ, PackedDistance} |
| `NumUploads` | `uint` | アップロードエントリ数 |

**スレッドグループ**: `[numthreads(THREADGROUP_SIZE, 1, 1)]`  
1スレッドが1ボクセルの書き込みを担当。

---

## DownsampleMeshSDFCS

**ファイル**: `DistanceFieldDownsampling.usf`

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `SourceMip` | `Texture3D<float>` | 上位解像度 Mip |
| `RWDestMip` | `RWTexture3D<float>` | 書き込み先低解像度 Mip |
| `DestMipResolution` | `uint3` | 出力解像度 |

**スレッドグループ**: `[numthreads(4, 4, 4)]`  
3D ボクセルグリッドで1スレッド1出力ボクセル。

---

## FDFObjectData（DistanceFieldLightingShared.ush）

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `LocalToWorld` | `float4x3` | ローカル→ワールド変換 |
| `InvBoxExtent` | `float3` | 逆ボックス範囲（UV 変換用）|
| `LocalPositionExtent` | `float3` | ローカル空間 AABB 範囲 |
| `UVScaleAndVolumeScale` | `float4` | Atlas UV スケール（xyz）+ ボリュームスケール（w）|
| `UVAdd` | `float3` | Atlas UV オフセット |
| `SelfShadowBias` | `float` | セルフシャドウ回避バイアス |
| `bMostlyTwoSided` | `uint` | 両面メッシュフラグ |
| `MinMaxDrawDistance2` | `float2` | 描画距離範囲（二乗値）|

---

## FDFObjectBounds（DistanceFieldLightingShared.ush）

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `Center` | `FDFVector3` | ワールド空間バウンド中心 |
| `SphereRadius` | `float` | バウンドスフィア半径 |

---

## DistanceToNearestSurfaceForObject

`MeshDistanceFieldCommon.ush`

```hlsl
float DistanceToNearestSurfaceForObject(
    FDFObjectData ObjectData,
    FDFVector3 WorldPosition,
    float MaxDistance);
```

---

## Atlas テクスチャ仕様

| 項目 | 値 |
|------|----|
| テクスチャ型 | `Texture3D` |
| フォーマット | `R16F`（デフォルト）または `R8`（メモリ節約時）|
| 格納値 | 正規化済み符号付き距離（`[-1, 1]` → 実距離に換算）|
| Mip 数 | `DistanceField::NumMips`（通常 3–4 段）|

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.DistanceFields.MaxPerMeshResolution` | メッシュあたり最大解像度 |
| `r.DistanceFields.DefaultVoxelDensity` | デフォルトボクセル密度 |
| `r.DistanceFields.AtlasAllocSize` | Atlas アロケーションサイズ |
| `r.DistanceFields.ForceAtlasFormat` | フォーマット強制（0=自動, 1=R16F, 2=R8）|
