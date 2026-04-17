# HZB Build シェーダー詳細

- グループ: a - HZB Build
- GPU 概要: [[01_hzb_gpu_overview]]
- CPU 詳細: [[a_hzb_build]]
- リファレンス: [[ref_hzb_build]]

---

## 概要

SceneDepth テクスチャから **Hierarchical Z-Buffer（ミップチェーン）** を生成する。  
Compute Shader で GroupShared メモリを活用し、1 Dispatch で最大 4 Mip を一度に書き出す。  
FurthestHZB（最大深度 = 最遠点）と ClosestHZB（最小深度 = 最近点）を同時生成する。

---

## レンダリングパスの構成

```
HZB Build Dispatch
  │
  ├─ [通常] ComputeShader パス（推奨）
  │   CS: HZB.usf#HZBBuildCS()
  │   GroupShared を用いた Parallel Reduce
  │   DIM_MIP_LEVEL_COUNT = 1〜4（1 Dispatch で複数 Mip）
  │
  └─ [フォールバック] PixelShader パス
      PS: HZB.usf#HZBBuildPS()
      Gather4 / SampleLevel による 2×2 Reduce
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `ParentTextureMip` | 前 Mip の深度テクスチャ（初回は SceneDepth）|
| `DispatchThreadIdToBufferUV` | スレッドID → BufferUV 変換係数 |
| `InvSize` | テクスチャの逆解像度 |
| `PixelViewPortMinMax` | ビューポート境界（POT パディング境界でのクランプ用）|
| `DIM_FURTHEST` / `DIM_CLOSEST` | 出力フラグ（どちらか/両方）|

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| `FurthestHZBOutput_0〜3` | `RWTexture2D<float>` | FurthestHZB 各 Mip（Min Reduce）|
| `ClosestHZBOutput_0〜3` | `RWTexture2D<float>` | ClosestHZB 各 Mip（Max Reduce + fp16 切り上げ）|

---

## シェーダーコアロジック

### FurthestHZB vs ClosestHZB

```hlsl
// Reverse-Z 環境での HZB セマンティクス:
//   FurthestHZB = Min(DeviceZ) → 最も遠い点（Reverse-Z では小さい値）
//   ClosestHZB  = Max(DeviceZ) → 最も近い点（fp16 に切り上げて保守的）

float RoundUpF16(float DeviceZ)
{
    // ClosestDeviceZ は fp16 に切り上げて保守的にする
    return f16tof32(f32tof16(DeviceZ) + 1);
}
```

### CS での GroupShared Reduce

```hlsl
groupshared uint  SharedMinDeviceZ[GROUP_TILE_SIZE * GROUP_TILE_SIZE];
groupshared float SharedMaxDeviceZ[GROUP_TILE_SIZE * GROUP_TILE_SIZE];

[numthreads(GROUP_TILE_SIZE, GROUP_TILE_SIZE, 1)]
void HZBBuildCS(uint2 GroupId : SV_GroupID, uint GroupThreadIndex : SV_GroupIndex)
{
    // 1. 入力深度を Gather4 で 4 テクセル読み取り
    float4 DeviceZx4 = Gather4(ParentTextureMip, ...);

    // 2. Min/Max を GroupShared に書き込み
    SharedMinDeviceZ[GroupThreadIndex] = asuint(min4(DeviceZx4));  // Furthest
    SharedMaxDeviceZ[GroupThreadIndex] = max4(DeviceZx4);           // Closest

    GroupMemoryBarrierWithGroupSync();

    // 3. 各 Mip レベルで 2x2 Reduce
    for (uint MipLevel = 1; MipLevel < DIM_MIP_LEVEL_COUNT; MipLevel++)
    {
        // GroupShared での Min/Max Reduce
        // → OutputMipLevel() で各 RWTexture に書き込み
    }
}
```

### Nanite VisBuffer 対応

```hlsl
// VIS_BUFFER_FORMAT に応じて深度の読み方が変わる
// VIS_BUFFER_FORMAT = 1: uint2（64bit VisBuffer）の .g チャンネル
// VIS_BUFFER_FORMAT = 4: 通常の SceneDepth（R32F）
// Froxel (DIM_FROXELS) 有効時は FroxelBuild.ush でボリュームフォグの Z スライスも並行生成
```

---

## CPU 呼び出しの流れ

```
BuildHZB()                           // SceneHZB.cpp
  │
  ├─ [Mip 0]
  │   AddPass(RDG_EVENT_NAME("HZBBuild Mip0"))
  │     Dispatch HZBBuildCS() または HZBBuildPS()
  │     入力: SceneDepth
  │     出力: FurthestHZB.Mip0 + ClosestHZB.Mip0
  │
  ├─ [Mip 1〜N] バッチ（DIM_MIP_LEVEL_COUNT = 2〜4 で複数 Mip を 1 Dispatch）
  │   AddPass(RDG_EVENT_NAME("HZBBuild Mip%d-%d"))
  │     入力: 前 Mip
  │     出力: FurthestHZB.Mip1〜N + ClosestHZB.Mip1〜N
  │
  └─ View.HZB（FSceneHZB）に登録 → SSR/Lumen/Nanite が参照
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `HZB.usf` | CS/PS | HZB ミップチェーン生成本体 |
| `ReductionCommon.ush` | ヘッダ | `InitialTilePixelPositionForReduction2x2()` など Reduce ユーティリティ |
| `Froxel/FroxelBuild.ush` | ヘッダ | DIM_FROXELS 時の Volumetric Fog Z スライス並行生成 |
| `Nanite/NaniteHZBCull.ush` | ヘッダ | Nanite HZB カリングの共通関数（`IsVisibleHZB()` 等）|
