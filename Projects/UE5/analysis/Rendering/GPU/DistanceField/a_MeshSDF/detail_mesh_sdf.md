# GPU a: Mesh SDF — Atlas アップロード・Mip ダウンサンプリング

- シェーダー: `DistanceFieldDownsampling.usf`, `DistanceFieldStreaming.usf`
- CPU 対応: [[a_mesh_sdf]] | [[ref_df_atlas]]
- 上位: [[01_distance_field_gpu_overview]]

---

## 概要

Mesh SDF の GPU アップロードパスは2段階からなる。  
1. `ScatterUploadCS`: CPU で生成された SDF ボクセルデータを Atlas テクスチャへ書き込む  
2. `DownsampleMeshSDFCS`: 高解像度 Mip を低解像度 Mip へダウンサンプリングする

---

## Atlas テクスチャ形式

```
Atlas Texture（3D テクスチャ）
  フォーマット: R16F（半精度浮動小数点）または R8（8bit 正規化）
               → r.DistanceFields.AtlasAllocSize と精度要件で決定

  各メッシュの SDF ボクセルが UVW 空間で矩形ブロックとして配置される
  AtlasAllocation = (UVW オフセット, UVW スケール) でアクセス
```

---

## ScatterUpload CS（ScatterUploadCS）

`DistanceFieldStreaming.usf`

```hlsl
// Scatter-as-Gather パターン:
// CPU が用意した {DestIndex, Value} ペアを元に
// Atlas テクスチャの指定位置へ書き込む Compute Shader

[numthreads(THREADGROUP_SIZE, 1, 1)]
void ScatterUploadCS(uint ThreadId : SV_DispatchThreadID)
{
    uint UploadIndex = ThreadId;
    // UploadBuffer から (DestX, DestY, DestZ, Distance) を読み込んで
    // RWAtlasTexture[DestPos] に書き込む
}
```

**特徴:**
- CPU で差分リスト（変更されたボクセルのみ）を生成
- GPU 側でスキャッタ書き込み → アトラス全体をコピーしない
- `FDistanceFieldSceneData::UpdateDistanceFieldAtlas()` から Dispatch

---

## Mip ダウンサンプリング CS（DownsampleMeshSDFCS）

`DistanceFieldDownsampling.usf`

```hlsl
// 高解像度 SDF Mip から低解像度 Mip を生成する
// 最近傍距離の最小値でダウンサンプリング（符号を保持）

[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, THREADGROUP_SIZE)]
void DownsampleMeshSDFCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    uint3 DstVoxel = DispatchThreadId;
    // 2x2x2 の上位 Mip テクセルを読み取り、min() でダウンサンプリング
    // 符号付き距離を保存するため単純 min は非推奨 → 絶対値 min 後に符号復元
}
```

---

## データフロー

```
CPU 側（FDistanceFieldSceneData::UpdateDistanceFieldAtlas）
  │
  ├─ PendingAddOperations: 新規 Mesh SDF データを UploadBuffer にセット
  │
  └─ RDG Pass を追加:
       ScatterUploadCS ──────────────────────→ RWAtlasTexture（Mip0 書き込み）
                                                │
       DownsampleMeshSDFCS ────────────────────→ RWAtlasTexture（Mip1, 2, ... 書き込み）
```

---

## オブジェクトバッファ（Object SDF Buffer）

Atlas アクセスのための位置情報は Object SDF バッファで管理される。

```hlsl
// DistanceFieldLightingShared.ush
struct FDFObjectData
{
    float4x3  LocalToWorld;           // ローカル→ワールド変換
    float3    InvBoxExtent;           // アトラス UV 変換用逆ボックス範囲
    float3    LocalPositionExtent;    // ローカル空間の範囲
    float4    UVScaleAndVolumeScale;  // アトラス UV スケール
    float3    UVAdd;                  // アトラス UV オフセット
    float     SelfShadowBias;
    uint      bMostlyTwoSided;        // 両面フラグ
    float2    MinMaxDrawDistance2;    // 描画距離の二乗
};

FDFObjectData LoadDFObjectData(uint ObjectIndex);
FDFObjectBounds LoadDFObjectBounds(uint ObjectIndex);
```

---

## DistanceToNearestSurface

`MeshDistanceFieldCommon.ush`

```hlsl
// Mesh SDF Atlas をサンプリングして最近傍サーフェス距離を返す
float DistanceToNearestSurfaceForObject(
    FDFObjectData ObjectData,
    FDFVector3 WorldPosition,
    float MaxDistance)
{
    // 1. WorldPosition → ローカル空間変換
    // 2. Atlas UVW 計算（UVAdd + LocalPos * UVScaleAndVolumeScale）
    // 3. Texture3D Atlas をサンプリング → 距離値を返す
}
```

---

## 主要パラメータ

| パラメータ | 説明 |
|-----------|------|
| `AtlasSizeX/Y/Z` | Atlas テクスチャ全体サイズ |
| `NumSceneObjects` | GPU 上の有効オブジェクト数 |
| `UploadBuffer` | ScatterUpload 用の {宛先インデックス, 距離値} リスト |
