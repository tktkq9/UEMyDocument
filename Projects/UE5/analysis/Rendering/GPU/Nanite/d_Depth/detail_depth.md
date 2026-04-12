# Nanite Depth シェーダー詳細

- グループ: d - Depth
- GPU 概要: [[01_nanite_gpu_overview]]
- CPU 詳細: [[a_nanite_cull_raster]]
- リファレンス: [[ref_depth]]

---

## 概要

Nanite の深度関連シェーダー。VisBuffer64 に書き込まれた深度値をハードウェア深度バッファと  
HTile（深度圧縮メタデータ）に転送する。  
また `NaniteDepthDecode.usf` は VSM・デバッグ用に HTile から深度値を復元する。

---

## DepthExport のコアロジック

`DepthExport` が Nanite のメインパスで呼ばれる。VisBuffer64 → SceneDepth + HTile の変換を担う。

```
VisBuffer64（depth packed as uint24）
      ↓
DepthExport（NaniteDepthExport.usf）
      ├─ SceneDepth（RWTexture2D<float>）  … ハードウェア深度バッファ
      ├─ SceneStencil（RWTexture2D<uint>） … ステンシルバッファ
      ├─ SceneHTile（RWStructuredBuffer）  … HTile 深度圧縮タイルデータ
      └─ ShadingMask（SHADING_MASK_EXPORT 時）… ピクセル分類マスク
```

### DepthExport のコアロジック詳細

```hlsl
[numthreads(HTILE_PIXELS_PER_TILE_WIDE, HTILE_PIXELS_PER_TILE_TALL, 1)]
void DepthExport(uint2 GroupId : SV_GroupID, uint ThreadIndex : SV_GroupIndex)
{
    // 1. スレッドをピクセル座標にマップ（Z-Order スウィズル）
    const uint2 PixelPos = SwizzleThreadIndex(GroupId, ThreadIndex);

    // 2. VisBuffer64 から深度 + クラスター情報を展開
    UnpackVisPixel(VisBuffer64[PixelPos], DepthInt, VisibleClusterIndex, TriIndex);

    // 3. Nanite ピクセルが存在しない HTile タイルは従来の SceneDepth をそのまま保持
    if (!WaveActiveAnyTrue(VisibleClusterIndex != 0xFFFFFFFF))
        return;  // タイル全体が非 Nanite → スキップ

    // 4. Nanite ピクセルの深度を SceneDepth に書き込み
    float NaniteDepthValue = asfloat(DepthInt);
    if (VisibleClusterIndex != 0xFFFFFFFF)
    {
        SceneDepth[PixelPos] = NaniteDepthValue;
        bWriteDepthStencil = true;
    }

    // 5. Wave 内で MinDepth / MaxDepth を集計 → HTile エンコード
    float TileMinDepth = WaveActiveMin(NaniteDepthValue);
    float TileMaxDepth = WaveActiveMax(NaniteDepthValue);
    SceneHTile[TileIndex] = EncodeTileMinMaxDepth(TileMinDepth, TileMaxDepth, PlatformConfig);

    // 6. ShadingMask: 非 Nanite ピクセルは 0（=非 Nanite フラグ）を書き込み
    //    Nanite ピクセルは ShadingBin + フラグをパック
    if (SHADING_MASK_EXPORT)
        ShadingMask[PixelPos] = PackEncodedMask(ShadingBin, ...);
}
```

### HTile の役割

HTile は GPU の深度バッファ圧縮メタデータ。タイル単位（8×8 px）の MinZ / MaxZ を保持し、  
その後の深度テスト（Z Cull）を高速化する。  
Nanite の SW ラスタライザは HTile を直接扱えないため、`DepthExport` が明示的に HTile を更新する必要がある。

---

## DepthDecode のコアロジック

`DepthDecode` は主に **Virtual Shadow Map（VSM）** やデバッグビュー用。  
HTile から深度値と圧縮レイアウト情報を復元して RWTexture2D に書き出す。

```hlsl
[numthreads(8, 8, 1)]
void DepthDecode(uint2 GroupId : SV_GroupID, uint ThreadIndex : SV_GroupIndex)
{
    // HTile バッファから該当タイルのエンコード値を読み取り
    const uint SceneHTileEnc = SceneHTileBuffer[TileIndex];

    // タイルの MinZ / MaxZ を 14bit 固定小数点でデコード
    const uint2 SceneTileMinMaxFP = DecodeTileMinMax(SceneHTileEnc, HiStencil, CompareMinZ);
    const float2 SceneTileMinMax  = SceneTileMinMaxFP / (float2)((1 << 14) - 1);

    // ピクセル単位の深度値をタイル圧縮から復元
    SceneZDecoded[PixelPos] = DecompressDepthValue(
        SceneDepth, SceneTileZMask, PixelPos, ThreadIndex, 0.0f, PlatformConfig);

    // レイアウト情報（Raw, MinMax packed, ZMask, TileIndex）をデバッグ出力
    SceneZLayout[PixelPos] = uint4(SceneHTileEnc, f32tof16(...), SceneTileZMask, TileIndex);
}
```

`DepthDecode` は `PLATFORM_SUPPORTS_HTILE_LOOKUP` が有効な GPU のみでコンパイルされる。

---

## 2つのシェーダーの使い分け

| シェーダー | 呼ばれるタイミング | 目的 |
|-----------|----------------|------|
| `DepthExport` | Nanite メインパス最後 | VisBuffer → ハードウェア SceneDepth + HTile 書き込み |
| `DepthDecode` | VSM / デバッグビューパス | HTile → 人間可読な深度テクスチャへ展開 |

---

## 入出力

### DepthExport

| リソース | 方向 | 内容 |
|---------|------|------|
| `VisBuffer64` | 読み | Nanite VisBuffer（depth + cluster + tri）|
| `SceneDepth`（RW）| 書き | ハードウェア SceneDepth バッファ |
| `SceneStencil`（RW）| 書き | ハードウェアステンシルバッファ |
| `SceneHTile`（RW）| 書き | HTile 深度圧縮メタデータ（タイル単位）|
| `ShadingMask`（RW）| 書き | SHADING_MASK_EXPORT 時: ピクセル分類マスク |
| `Velocity`（RW）| 書き | VELOCITY_EXPORT 時: Motion Vector |

### DepthDecode

| リソース | 方向 | 内容 |
|---------|------|------|
| `SceneHTileBuffer` | 読み | HTile バッファ（入力）|
| `SceneDepth` | 読み | 既存の SceneDepth（Zマスク圧縮解除に使用）|
| `SceneZDecoded`（RW）| 書き | 復元された深度値テクスチャ |
| `SceneZLayout`（RW）| 書き | デバッグ用深度レイアウト情報（uint4）|

---

## CPU 呼び出しの流れ

```
Nanite::AddDepthExportPass()              // NaniteCullRaster.cpp
  └─ DepthExport CS ディスパッチ
       → SceneDepth + SceneHTile + ShadingMask 書き込み

Nanite::AddDepthDecodePass()              // NaniteCullRaster.cpp（VSM / デバッグ用）
  └─ DepthDecode CS ディスパッチ
       → SceneZDecoded + SceneZLayout 書き込み
```
