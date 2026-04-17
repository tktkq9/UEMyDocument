# REF: HZB Build シェーダー

- グループ: a - HZB Build
- 詳細: [[detail_hzb_build]]
- CPU リファレンス: [[ref_hzb_resources]]
- ソース: `Engine/Shaders/Private/HZB.usf`

---

## HZBBuildCS（HZB.usf:216）

### エントリポイント

```hlsl
[numthreads(GROUP_TILE_SIZE, GROUP_TILE_SIZE, 1)]
void HZBBuildCS(
    uint2 GroupId           : SV_GroupID,
    uint  GroupThreadIndex  : SV_GroupIndex)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Compute Shader |
| **スレッドグループ** | 8×8 = 64 threads |
| **目的** | 親 Mip から 1〜4 Mip を GroupShared Reduce で生成 |
| **GroupShared** | `SharedMinDeviceZ[64]`（uint / Min）, `SharedMaxDeviceZ[64]`（float / Max）|

#### パーミュテーション

| ディメンション | 説明 |
|-------------|------|
| `DIM_MIP_LEVEL_COUNT` | 1 Dispatch で出力する Mip 数（1〜4）|
| `DIM_FURTHEST` | FurthestHZB を出力するか |
| `DIM_CLOSEST` | ClosestHZB を出力するか |
| `DIM_FROXELS` | Volumetric Fog Froxel を並行生成するか |
| `VIS_BUFFER_FORMAT` | 入力深度フォーマット（0〜4）|

#### CPU バインド

```cpp
// SceneHZB.cpp - BuildHZB()
FHZBBuildParameters* PassParameters = GraphBuilder.AllocParameters<FHZBBuildParameters>();
PassParameters->DispatchThreadIdToBufferUV = ...;
PassParameters->InvSize = FVector2f(1.0f / Width, 1.0f / Height);
PassParameters->ParentTextureMip = ParentMipSRV;
PassParameters->FurthestHZBOutput_0 = GraphBuilder.CreateUAV(FurthestHZB, Mip);
PassParameters->ClosestHZBOutput_0  = GraphBuilder.CreateUAV(ClosestHZB, Mip);

GraphBuilder.AddPass(
    RDG_EVENT_NAME("HZBBuildCS"),
    PassParameters,
    ERDGPassFlags::Compute,
    [PassParameters, NumGroupsX, NumGroupsY](FRHICommandList& RHICmdList)
    {
        FComputeShaderUtils::Dispatch(RHICmdList, ComputeShader, *PassParameters,
            FIntVector(NumGroupsX, NumGroupsY, 1));
    });
```

#### 出力リソース

| リソース | 型 | 用途 |
|---------|------|------|
| `FurthestHZBOutput_0〜3` | `RWTexture2D<float>` | Min-Reduce ミップ（最遠深度）|
| `ClosestHZBOutput_0〜3` | `RWTexture2D<float>` | Max-Reduce ミップ（最近深度 + fp16 切り上げ）|

---

## HZBBuildPS（HZB.usf:344）

### エントリポイント

```hlsl
void HZBBuildPS(
    float4 SvPosition : SV_POSITION,
    out float4 OutColor : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Pixel Shader |
| **目的** | CS が使えない場合の PS フォールバック。Gather4 で 2×2 Min Reduce |
| **使用条件** | `DIM_USE_MIPINDEX = 0`（PS では `SourceMipIndex` uniform を使用）|
| **出力** | `OutColor.r` = Min DeviceZ（FurthestHZB 相当）|

---

## 主要ユーティリティ（HZB.usf）

### Gather4 関数

```hlsl
float4 Gather4(Texture2D Texture, SamplerState TextureSampler, float2 BufferUV)
// 2×2 テクセルを GatherRed（またはバイリニアサンプル 4 回）で読み取る
// InputViewportMaxBound でビューポート境界をクランプ
```

### OutputMipLevel

```hlsl
void OutputMipLevel(uint MipLevel, uint2 OutputPixelPos, float FurthestDeviceZ, float ClosestDeviceZ)
// DIM_MIP_LEVEL_COUNT に応じてコンパイル時展開
// MipLevel 0 = OutputMipLevel 呼び出し側が直接書き込み
// MipLevel 1〜3 = 本関数経由で RWTexture_N に書き込み
```

### ConvertFromDeviceZ / ConvertToDeviceZ

```hlsl
float ConvertFromDeviceZ(float DeviceZ, float4 InvDeviceZToWorldZTransform)
// Ortho / Perspective 両対応のデバイスZ → ワールドZ変換

float ConvertToDeviceZ(float SceneDepth, float4 InvDeviceZToWorldZTransform)
// 逆変換
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.HZBOcclusion` | 1 | HZB Occlusion Query 有効 |
| `r.DownsampledOcclusionQueries` | 1 | 半解像度で HZB を生成 |
| `r.HZBOcclusion.SkipLastFrameOutdatedTests` | 1 | 古いフレームの結果を無効化 |

---

> [!note]- FurthestHZB と ClosestHZB の使い分け
> - **FurthestHZB**（Min-Reduce）: SSR・Lumen Screen Trace が「この方向に物体があるか」を判断するのに使う。小さな値 = 遠い物体がある  
> - **ClosestHZB**（Max-Reduce）: Occlusion Query で「遮蔽されているか」を判断するのに使う。大きな値 = 近くに遮蔽物がある  
> - fp16 切り上げは「ClosestHZB は保守的でなければならない」という要件から。切り上げることでオクルージョンの誤判定（本来は見えるのに隠れると判断）を防ぐ
