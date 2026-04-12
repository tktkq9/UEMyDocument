# REF: Lumen Card Capture シェーダー

- グループ: a - Surface Cache
- 詳細: [[detail_card_capture]]
- CPU リファレンス: [[ref_lumen_scene]]
- ソース: `Engine/Shaders/Private/Lumen/LumenCardVertexShader.usf`  
          `Engine/Shaders/Private/Lumen/LumenCardPixelShader.usf`  
          `Engine/Shaders/Private/Lumen/LumenCardComputeShader.usf`  
          `Engine/Shaders/Private/Lumen/SurfaceCache/LumenSurfaceCache.usf`

---

## LumenCardVertexShader.usf

### エントリポイント

```hlsl
void Main(
    FVertexFactoryInput Input,
    out FLumenCardVSToPS Output)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Vertex Shader |
| **目的** | Card ジオメトリを Clip 空間に変換して PS へ補間値を渡す |
| **WPO** | 意図的に無効化（コメントアウト）— SDF との整合性のため |
| **出力構造体** | `FLumenCardVSToPS`（`FactoryInterpolants` + `Position`） |

#### CPU バインド

```cpp
// LumenSceneRendering.cpp
// FLumenCardPassUniformParameters → シェーダー側で #define SceneTexturesStruct LumenCardPass.SceneTextures
GraphBuilder.AddPass(RDG_EVENT_NAME("LumenCardCapture"),
    PassParameters, ERDGPassFlags::Raster,
    [](FRHICommandList& RHICmdList) {
        // DrawMeshCommands → VS = LumenCardVertexShader
    });
```

---

## LumenCardPixelShader.usf

### エントリポイント

```hlsl
void Main(
    FVertexFactoryInterpolantsVSToPS Interpolants,
    FLumenCardInterpolantsVSToPS PassInterpolants,
    in float4 SvPosition : SV_Position,
    OPTIONAL_IsFrontFace,
    out float4 OutTarget0 : SV_Target0,  // Albedo + Opacity
    out float4 OutTarget1 : SV_Target1,  // Normal + Roughness
    out float4 OutTarget2 : SV_Target2)  // Emissive
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Pixel Shader |
| **目的** | マテリアルグラフを評価して Surface Cache 3RT に書き込む |
| **マクロ** | `LUMEN_CARD_CAPTURE=1`（マテリアル側の分岐制御）|
| **共通処理** | `LumenCardBasePass.ush#LumenCardBasePass()` に委譲 |

#### 出力

| ターゲット | チャンネル | 内容 |
|---------|---------|------|
| `OutTarget0` | `rgba` | `float4(DiffuseColor.rgb, Opacity)` |
| `OutTarget1` | `rgba` | `float4(EncodeNormal(Normal), Roughness)` |
| `OutTarget2` | `rgba` | `float4(Emissive.rgb, 0)` |

---

## LumenCardComputeShader.usf（Nanite 専用）

### エントリポイント

```hlsl
[numthreads(64, 1, 1)]
void Main(
    uint ThreadIndex : SV_GroupIndex,
    uint GroupID : SV_GroupID)
```

| 項目 | 内容 |
|-----|------|
| **タイプ** | Compute Shader |
| **目的** | Nanite VisibilityBuffer から Card ピクセルを復元してアトラスに書き込む |
| **スレッドグループ** | 64 threads × 8×8 ピクセルタイル（Morton デコードで配置）|
| **入力** | `NaniteShading.VisBuffer64`・`NaniteShading.InViews`・`NaniteShading.ShadingBinData` |
| **出力** | `LumenCardOutputs.OutTarget0/1/2`（RWTexture2D 経由）|

#### ルートコンスタント

| 番号 | 名前 | 説明 |
|-----|------|------|
| `GetRootConstant0()` | `ShadingBin` | 処理対象のシェーディングビン番号 |
| `GetRootConstant1()` | `QuadBinning` | Quad binning モードか否か |
| `GetRootConstant2()` | `MeshPassIndex` | メッシュパスのインデックス |

#### 実行フロー

```
Main()
  ├─ PackedTile → CardIndex, TopLeft を復元
  ├─ MortonDecode(ThreadIndex) → PixelPos
  ├─ VisBuffer64[PixelPos] → VisibleClusterIndex, TriIndex
  ├─ ShadingBin 一致確認
  └─ ProcessPixel() → LumenCardBasePass() → ExportPixel()
                       → RWOutTarget0/1/2[PixelPos] に書き込み
```

#### CPU バインド

```cpp
// Nanite::DispatchShading() 内
FNaniteShadingCommand Cmd;
Cmd.ComputeShader = LumenCardComputeShader;   // IMPLEMENT_GLOBAL_SHADER で登録
Cmd.BoundShaderState = ...;
// DispatchComputeShader(RHICmdList, ComputeShader, GroupCount.X, 1, 1)
```

---

## LumenSurfaceCache.usf（後処理 CS 群）

### CopyCapturedCardPageCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void CopyCapturedCardPageCS(uint3 GroupId : SV_GroupID, uint3 GroupThreadId : SV_GroupThreadID, uint GroupThreadIndex : SV_GroupIndex)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Card Capture の結果（キャプチャアトラス）を物理ページアトラスにコピー |
| **入力** | `AlbedoCardCaptureAtlas`, `NormalCardCaptureAtlas`, `EmissiveCardCaptureAtlas` |
| **出力** | `RWAlbedoAtlas`, `RWNormalAtlas`（UNORM float4）, `RWDepthAtlas` |
| **CPU 関数** | `LumenSceneRendering.cpp#CopyCardCaptureLightingToAtlas()` |

### ClearCompressedAtlasCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void ClearCompressedAtlasCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 不要になったアトラスページを 0 クリア |
| **出力** | `RWAtlasBlock4`（uint4 圧縮フォーマット）/ `RWAtlasBlock2`（uint2）|

### GenerateDilationTileDataCS

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void GenerateDilationTileDataCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | ページ境界でのサンプリングアーティファクトを防ぐためのダイレーション処理のタイルデータ生成 |

### LumenCardCopyPS

```hlsl
void LumenCardCopyPS(
    float2 InUV : TEXCOORD0,
    float4 SvPosition : SV_Position,
    out float4 OutColor0 : SV_Target0)
```

| 項目 | 内容 |
|-----|------|
| **目的** | キャプチャアトラスの直接コピー（フルスクリーン三角形）|

### LumenCardResamplePS

```hlsl
void LumenCardResamplePS(
    float2 InUV : TEXCOORD0,
    float4 SvPosition : SV_Position,
    out float4 OutAlbedoOpacity : SV_Target0,
    out float4 OutNormal        : SV_Target1,
    out float4 OutEmissive      : SV_Target2)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Surface Cache の解像度が変化した際にアトラスを再サンプリング |
| **CPU 関数** | `ResampleLightingHistoryToCardCaptureAtlasCS()` 経由 |

---

## LumenCardBasePass.ush（共通シェーディングヘッダ）

### FLumenCardBasePassOutput 構造体

```hlsl
struct FLumenCardBasePassOutput
{
    float4 Target0;  // Albedo + Opacity
    float4 Target1;  // Normal + Roughness
    float4 Target2;  // Emissive
};
```

### LumenCardBasePass 関数

```hlsl
FLumenCardBasePassOutput LumenCardBasePass(
    FPixelMaterialInputs PixelMaterialInputs,
    FMaterialPixelParameters MaterialParameters,
    float4 SvPosition)
```

| 項目 | 内容 |
|-----|------|
| **目的** | PS/CS 両方から共通で使えるマテリアル評価・出力パッキング関数 |
| **Substrate 対応** | `TEMPLATE_USES_SUBSTRATE` 時は BSDF ツリーを展開して Diffuse/Specular を平均化 |
| **Specular** | Lumen は Diffuse のみ保存 — Specular は GGX として Diffuse に折り畳む |

---

## 主要 CVar / パーミュテーション

| CVar / マクロ | デフォルト | 説明 |
|-------------|---------|------|
| `LUMEN_CARD_CAPTURE` | 1（パス専用）| Surface Cache キャプチャ専用コードパスを有効化 |
| `IS_NANITE_PASS` | 0/1 | CS 版は必ず 1（それ以外はコンパイルエラー）|
| `TEMPLATE_USES_SUBSTRATE` | 環境依存 | Substrate マテリアルシステム有効時 |
| `r.Lumen.SurfaceCache.AtlasSize` | 2048 | Surface Cache アトラスの解像度 |

---

> [!note]- Card Capture と通常 GBuffer パスの違い
> 通常の Base Pass は `SceneTextures` に書き込み最終ライティングに使うが、Card Capture は**Surface Cache アトラス専用の 3RT**（Albedo/Normal/Emissive）に書き込む別パスとして独立している。  
> マテリアルシェーダーは両方で共有コードを使うが、`LUMEN_CARD_CAPTURE=1` マクロで動作が切り替わる。  
> WPO を無効化するのは、Lumen の Mesh SDF はスタティックな形状で計算されており WPO の変形との乖離を防ぐため。

> [!note]- Nanite CS 版と通常 VS/PS 版の共存
> Nanite メッシュは `LumenCardComputeShader.usf` が担当し、通常メッシュは `LumenCardVertexShader.usf` + `LumenCardPixelShader.usf` が担当する。  
> 両者は `LumenCardBasePass.ush` の `LumenCardBasePass()` 関数を共有しており、出力フォーマット・チャンネルパッキングは同一。  
> Nanite CS 版は VisibilityBuffer から「誰がどのピクセルを占有しているか」を読み取り、同じシェーディングロジックをコンピュートシェーダーで再実行する方式。

> [!note]- Surface Cache アトラスのページ管理
> Surface Cache は大きな 2D テクスチャアトラスを使い、各 Card に物理ページを動的に割り当てる（Virtual Texture 的な仕組み）。  
> `LumenVirtualTextureCommon.ush` で物理ページアドレスの変換が定義されており、`CopyCapturedCardPageCS()` がキャプチャ結果を正しい物理ページ位置にコピーする。  
> 使われていないページは `ClearCompressedAtlasCS()` で定期的にクリアされ、アトラスの断片化を防ぐ。
