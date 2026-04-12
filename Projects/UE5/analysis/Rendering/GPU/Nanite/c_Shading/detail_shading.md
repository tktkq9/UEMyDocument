# Nanite Shading シェーダー詳細

- グループ: c - Shading
- GPU 概要: [[01_nanite_gpu_overview]]
- CPU 詳細: [[b_nanite_materials_shading]]
- リファレンス: [[ref_shading]]

---

## 概要

Nanite のマテリアルシェーディングパイプライン。VisBuffer64 が確定した後、  
**Shade Binning → GBuffer Export** の順で実行し、最終的に SceneColor/GBuffer を生成する。  
ピクセルをマテリアル別ビンに分類することで、マテリアル定数の切り替えを最小化する **遅延シェーディング** 設計。

---

## パス構成

```
Shade Binning（NaniteShadeBinning.usf）
  ├─ ShadingBinBuildCS   … VisBuffer ピクセルを ShadingBin に分類・カウント
  ├─ ShadingBinReserveCS … 各ビンのバッファ領域を確保・Indirect Args 生成
  └─ ShadingBinValidateCS … デバッグ用：書き込みカウント検証

GBuffer Export / Material Shading（NaniteExportGBuffer.usf）
  ├─ EmitSceneDepthPS       … VisBuffer → SceneDepth + ShadingMask + Velocity
  ├─ EmitSceneStencilPS     … ShadingMask からデカール受信ピクセルに Stencil を書く
  ├─ EmitCustomDepthStencilPS … Custom Depth/Stencil の書き出し
  ├─ EmitShadowMapPS        … Nanite PixelShader 経由のシャドウマップ出力
  ├─ EmitHitProxyIdPS       … エディタ Hit Proxy ID 書き込み
  └─ EmitEditorSelectionDepthPS … エディタ選択オブジェクトの Depth 書き込み
```

---

## Shade Binning のコアロジック

### ShadingBinBuildCS — VisBuffer ピクセルをビンに分類

```hlsl
[numthreads(SHADING_BIN_TILE_THREADS, 1, 1)]
void ShadingBinBuildCS(uint ThreadIndex : SV_GroupIndex, uint2 GroupId : SV_GroupID)
{
    // タイル座標 → Z-Order カーブでスレッドをピクセルにマップ
    uint2 Coord = GroupId.xy * SHADING_BIN_TILE_SIZE;
    Coord += ZOrder2D(ThreadIndex, 3);

    // Single Wave 対応 GPU では Wave 内スカラライズで高効率処理
    if (bSingleWave)
        BinShadingQuad<true>(Coord, ThreadIndex);
    else
        BinShadingQuad<false>(Coord, ThreadIndex);
}
```

`BinShadingQuad()` の内部処理：
1. `VisBuffer64` から VisibleClusterIndex + TriIndex を読み出し
2. クラスター情報から `ShadingBin`（マテリアルビン番号）を決定
3. 同一 Wave 内で同じビンのスレッドを **スカラライズ**（`ScalarizeForEachMatching`）してまとめて処理
4. Quad（2×2 ピクセル）単位でフルタイル / ルーズ要素に分類し、ビンカウントをアトミックインクリメント

### ShadingBinReserveCS — ビンのバッファ領域確保

```hlsl
[numthreads(64, 1, 1)]
void ShadingBinReserveCS(uint ShadingBin : SV_DispatchThreadID)
{
    const uint BinPixelCount = GetShadingBinMeta(ShadingBin).ElementCount;

    // Indirect Dispatch 引数: ThreadGroupCountX = ceil(BinPixelCount / COMPUTE_MATERIAL_GROUP_SIZE)
    ShadingBinArgs.x = DivideAndRoundUp(BinPixelCount, COMPUTE_MATERIAL_GROUP_SIZE);
    OutShadingBinArgs.Store4(ShadingBin * 16u, ShadingBinArgs);
}
```

---

## GBuffer Export のコアロジック

### EmitSceneDepthPS — メインの深度書き出し

```hlsl
void EmitSceneDepthPS(
    in float4  SvPosition : SV_Position,
    out uint   OutShadingMask : SV_Target0,  // SHADING_MASK_EXPORT 時
    out float4 OutVelocity    : SV_Target1,  // VELOCITY_EXPORT 時
    out float  OutDepth       : SV_Depth
)
{
    UnpackVisPixel(VisBuffer64[PixelPos], DepthInt, VisibleClusterIndex, TriIndex);

    if (VisibleClusterIndex != 0xFFFFFFFF)
    {
        // ShadingBin 番号を VisBuffer から取得してパック
        OutShadingMask = PackShadingMask(ShadingBin, bIsDecalReceiver, ...);
        // Velocity: 前フレームとの変換差分で MotionVector 計算
        OutVelocity = CalcVelocity(CurClipPos, PrevClipPos);
        OutDepth = asfloat(DepthInt);  // VisBuffer の深度をそのまま出力
    }
    else { discard; }
}
```

### マテリアルシェーディング（DispatchBasePass）

`EmitSceneDepthPS` が `ShadingMask` を書いた後、CPU 側が各シェーディングビンに対して  
Indirect Draw を発行してマテリアル評価シェーダーを実行する。

```
ShadingBinMeta → Indirect Draw per Bin
  → BasePassシェーダー（マテリアルグラフ評価）
    → VisBuffer64 から VisibleCluster + TriIndex 取得
    → バリセントリック座標で頂点属性補間（UV, Normal, etc.）
    → マテリアルグラフ実行
    → GBuffer A/B/C/D + SceneColor（エミッシブ）書き込み
```

---

## ShadingMask のフォーマット

| ビット | 内容 |
|--------|------|
| 15-8 | ShadingBin 番号 |
| 7 | bIsNanitePixel |
| 6 | bIsDecalReceiver |
| 5-0 | その他フラグ（Substrate タイルタイプ等）|

`EmitSceneDepthPS` が書き込み、後続の `EmitSceneStencilPS` がデカール用ステンシルを立てるために読む。

---

## 入出力

### 入力

| リソース | 内容 |
|---------|------|
| `VisBuffer64` | ラスタライズ後の Visibility Buffer |
| `RasterBinMeta` | マテリアルビン別フラグ・容量 |
| `InstanceSceneData` / `ClusterPageData` | インスタンス・クラスター情報 |

### 出力（Shade Binning）

| リソース | 内容 |
|---------|------|
| `OutShadingBinData` | ビン別ピクセルリスト（Coord + Quad 情報）|
| `OutShadingBinArgs` | ビン別 Indirect Dispatch 引数 |

### 出力（GBuffer Export）

| リソース | 内容 |
|---------|------|
| `SceneDepth` | Nanite ピクセルの深度（SV_Depth）|
| `ShadingMask` | ピクセル分類マスク（ShadingBin 番号 + フラグ）|
| `OutVelocity` | Motion Vector（VELOCITY_EXPORT 時）|
| GBuffer A/B/C/D | マテリアル属性（DispatchBasePass 後）|

---

## CPU 呼び出しの流れ

```
Nanite::BuildShadingCommands()             // NaniteShading.cpp
  │
  ├─ ShadingBinBuildCS
  ├─ ShadingBinReserveCS
  └─ ShadingBinValidateCS（デバッグ時）

Nanite::DispatchBasePass()                 // NaniteShading.cpp
  │
  ├─ EmitSceneDepthPS（Indirect Draw — 全ビン1回）
  │    → SceneDepth + ShadingMask + Velocity 書き込み
  │
  └─ BasePass Material Shaders（Indirect Draw per Bin）
       → GBuffer A/B/C/D + SceneColor 書き込み
```
