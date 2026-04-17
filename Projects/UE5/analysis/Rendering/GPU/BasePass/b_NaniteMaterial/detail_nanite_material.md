# Nanite Material GBuffer シェーダー詳細

- グループ: b - NaniteMaterial
- GPU 概要: [[01_basepass_gpu_overview]]
- CPU 詳細: [[b_nanite_gbuffer]]
- リファレンス: [[ref_nanite_material]]

---

## 概要

Nanite メッシュは通常の VS/PS ではなく **VisibilityBuffer + ShadeBinning** を使って GBuffer を生成する。  
VisibilityBuffer（`R32G32_UINT`）には「どのインスタンスのどのトライアングルか」だけを記録し、  
後処理の Compute Shader がマテリアルを評価して GBuffer に書き込む（Visibility Shading）。

---

## レンダリングパスの構成

```
Nanite GBuffer 生成
  │
  ├─ [1] Nanite Rasterize（NaniteRasterizer.usf）
  │   → VisibilityBuffer（32bit: ClusterID + TriangleID）に書き込み
  │
  ├─ [2] ShadeBin 構築（NaniteShadeBinning.usf）
  │   ShadeBinBuildCS: VisibilityBuffer の各ピクセルをシェーディングビン別に分類
  │   ShadeBinReserveCS: 各 ShadingBin の開始オフセットを計算
  │
  ├─ [3] Export GBuffer（NaniteExportGBuffer.usf）
  │   EmitSceneDepthPS: VisibilityBuffer → SceneDepth に変換
  │   EmitSceneStencilPS: VisibilityBuffer → Stencil に変換
  │
  └─ [4] Material Shading（Nanite Compute Shading）
      Nanite Compute Shader（ShadingBin 別）:
        VisibilityBuffer → Cluster/Triangle 復元 → マテリアル評価 → GBuffer 書き込み
      （実体は LumenCardComputeShader と同じ Nanite Visibility Shading 方式）
```

---

## 入出力

### 入力

| リソース | 説明 |
|---------|------|
| `VisibilityBuffer` | `R32G32_UINT`: 上位32bit = DepthZ, 下位32bit = ClusterID + TriangleID |
| `ShadingBinData` | ShadingBin ごとのオフセット・カウント |
| `GPUScene.InstanceDataBuffer` | インスタンスの変換行列等 |

### 出力

| リソース | フォーマット | 内容 |
|---------|------------|------|
| `SceneDepth` | `R32F` | VisibilityBuffer から復元した深度 |
| `GBufferA/B/C/D/E` | 各 `R8G8B8A8` | マテリアル属性（法線・BaseColor・Roughness 等）|

---

## シェーダーコアロジック

### ShadeBinBuildCS（NaniteShadeBinning.usf:886）

```hlsl
[numthreads(SHADING_BIN_TILE_THREADS, 1, 1)]
void ShadeBinBuildCS(uint ThreadIndex : SV_GroupIndex, uint2 GroupId : SV_GroupID)
{
    // VisibilityBuffer の各ピクセルを読み取り
    uint64 VisData = VisibilityBuffer[PixelPos];
    uint ClusterId = UnpackClusterId(VisData);
    uint ShadingBin = GetShadingBinFromCluster(ClusterId);  // マテリアル ID が ShadingBin

    // 各 ShadingBin のピクセル数をカウント（Atomic）
    InterlockedAdd(ShadingBinPixelCount[ShadingBin], 1);
}
```

### EmitSceneDepthPS（NaniteExportGBuffer.usf:70）

```hlsl
void EmitSceneDepthPS(
    in float4 SvPosition : SV_POSITION,
    out float OutDepth : SV_Depth)
{
    // VisibilityBuffer からピクセルのデバイスZ を取得して SceneDepth に書き込み
    float2 UV = SvPosition.xy * InvSize;
    uint2 PixelPos = uint2(SvPosition.xy);

    uint64 VisData = InVisBuffer64.Load(PixelPos);
    float DeviceZ = asfloat(uint(VisData >> 32));  // 上位 32bit = Depth

    OutDepth = DeviceZ;
}
```

### Nanite Material Shading（Compute Shading）

```
NaniteShadeBinDispatch():
  ShadingBin ごとに 1 つの ComputeShader を Dispatch
  各スレッドが 1 ピクセルを処理:
    1. VisibilityBuffer → ClusterID + TriangleID を復元
    2. Cluster から頂点データを取得（Nanite Stream）
    3. バリセントリック座標で頂点属性を補間（UV/Normal/Tangent）
    4. マテリアルグラフを評価（BasePassPixelShader.usf 相当）
    5. GBuffer に書き込み（RWTexture2D UAV）
```

---

## CPU 呼び出しの流れ

```
Nanite::DispatchShading()                   // NaniteShading.cpp
  │
  ├─ BuildShadingCommands() → ShadeBin 別コマンドリスト生成
  ├─ AddPass(ShadeBinBuildCS)    → ShadingBin 分類
  ├─ AddPass(ShadeBinReserveCS)  → オフセット計算
  ├─ AddPass(EmitSceneDepthPS)   → SceneDepth 書き込み
  └─ AddPass(NaniteShadingCS)    × ShadingBin 数
      各 ShadingBin のピクセルに対してマテリアルを評価
      → GBuffer UAV に書き込み
```

---

## 関連シェーダー

| ファイル | 種別 | 役割 |
|---------|------|------|
| `Nanite/NaniteShadeBinning.usf` | CS | VisibilityBuffer → ShadingBin 分類 |
| `Nanite/NaniteExportGBuffer.usf` | PS | VisibilityBuffer → SceneDepth/Stencil 変換 |
| `Nanite/NaniteRasterizer.usf` | VS/PS/CS | VisibilityBuffer 生成（ラスタライズ）|
