# REF: Nanite Depth シェーダー

- グループ: d - Depth
- 詳細: [[detail_depth]]
- CPU リファレンス: [[ref_nanite_cull_raster]]
- ソース: `Engine/Shaders/Private/Nanite/NaniteDepthDecode.usf`
          `Engine/Shaders/Private/Nanite/NaniteDepthExport.usf`

---

## NaniteDepthDecode.usf

### DepthDecode（HTile Lookup 対応 GPU 版）

```hlsl
[numthreads(8, 8, 1)]
void DepthDecode(
    uint2 GroupId     : SV_GroupID,
    uint ThreadIndex  : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | GPU の HTile バッファから SceneDepth の圧縮済み深度値を復元し、デバッグ / VSM 用テクスチャに展開 |
| **条件** | `PLATFORM_SUPPORTS_HTILE_LOOKUP=1` のプラットフォームのみ |
| **入力** | `SceneHTileBuffer`（HTile）, `SceneDepth`, `HTileConfig`（プラットフォーム設定）, `ViewRect` |
| **出力** | `SceneZDecoded`（float）= ピクセル単位復元深度；`SceneZLayout`（uint4）= 圧縮メタデータ詳細 |
| **CPU 関数** | `Nanite::AddDepthDecodePass()` |

### DepthDecode（フォールバック版）

```hlsl
[numthreads(8, 8, 1)]
void DepthDecode(uint3 PixelPos : SV_DispatchThreadID, uint ThreadIndex : SV_GroupIndex)
```

| 項目 | 内容 |
|-----|------|
| **目的** | HTile Lookup 非対応プラットフォーム向けのフォールバック（実質 no-op）|
| **条件** | `PLATFORM_SUPPORTS_HTILE_LOOKUP=0` |

---

## NaniteDepthExport.usf

### DepthExport（HTile 書き込み版）

```hlsl
[numthreads(HTILE_PIXELS_PER_TILE_WIDE, HTILE_PIXELS_PER_TILE_TALL, 1)]
void DepthExport(
    uint2 GroupId    : SV_GroupID,
    uint ThreadIndex : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | VisBuffer64 の Nanite 深度値を SceneDepth + SceneHTile + ShadingMask に書き出す。SW ラスタライザが書けない HTile を明示的に更新する核心パス |
| **条件** | `PLATFORM_SUPPORTS_HTILE_LOOKUP=1` |
| **スレッドグループサイズ** | `HTILE_PIXELS_PER_TILE_WIDE` × `HTILE_PIXELS_PER_TILE_TALL`（通常 8×8）|
| **入力** | `VisBuffer64`, `SceneDepth`（既存値）, `SceneStencil`（既存値）, `RasterBinMeta`, `DepthExportConfig` |
| **出力** | `SceneDepth`（RW）, `SceneStencil`（RW）, `SceneHTile`（RW）, `ShadingMask`（RW, オプション）, `Velocity`（RW, オプション）|
| **CPU 関数** | `Nanite::AddDepthExportPass()` |

#### パーミュテーション

| マクロ | 値 | 説明 |
|--------|-----|------|
| `SHADING_MASK_EXPORT` | 0/1 | ShadingMask テクスチャへの同時書き込み |
| `VELOCITY_EXPORT` | 0/1 | Motion Vector テクスチャへの同時書き込み |
| `VIRTUAL_TEXTURE_TARGET` | 0/1 | VSM（Virtual Shadow Map）ターゲット向け |

### DepthExport（フォールバック版）

```hlsl
[numthreads(8, 8, 1)]
void DepthExport(uint3 PixelPos : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | HTile Lookup 非対応プラットフォーム向け — VisBuffer64 から SceneDepth への単純コピー |
| **条件** | `PLATFORM_SUPPORTS_HTILE_LOOKUP=0` |
| **出力** | `SceneDepth`（RW）のみ（HTile / ShadingMask は書かない）|

---

## 主要シェーダーパラメーター

| パラメーター | 型 | 内容 |
|------------|-----|------|
| `DepthExportConfig` | `uint4` | .x=プラットフォーム設定, .y=タイル幅, .z=Decalステンシル値, .w=最大可視クラスター数 |
| `VisBuffer64` | `Texture2D<UlongType>` | Nanite VisBuffer（入力）|
| `SceneHTile` | `RWStructuredBuffer<uint>` | HTile バッファ（直接書き込み）|
| `SceneDepth` | `RWTexture2D<float>` | ハードウェア深度バッファ |
| `SceneStencil` | `RWTexture2D<uint>` | ハードウェアステンシルバッファ |
| `ShadingMask` | `RWTexture2D<uint>` | ピクセル分類マスク（オプション）|
| `HTileConfig` | `uint4` | DepthDecode 用 HTile 設定（`DepthExportConfig` と同様の構造）|
| `SceneHTileBuffer` | `StructuredBuffer<uint>` | DepthDecode 用 HTile 入力 |
| `SceneZDecoded` | `RWTexture2D<float>` | DepthDecode 出力：復元深度値 |
| `SceneZLayout` | `RWTexture2D<uint4>` | DepthDecode 出力：圧縮メタデータ詳細 |

---

> [!note]- SW ラスタライザが HTile を直接書けない理由
> GPU の HTile は固定機能の深度圧縮ユニットが自動更新するが、Compute Shader（SW ラスタライザ）からは  
> `SV_Depth` を出力する機能がないため HTile の自動更新が行われない。  
> `DepthExport` では Wave 単位で 8×8 タイル内の MinZ / MaxZ を集計し、  
> `EncodeTileMinMaxDepth()` で HTile フォーマットにエンコードして手動書き込みする。

> [!note]- HTILE_PIXELS_PER_TILE の意味
> `HTILE_PIXELS_PER_TILE_WIDE` / `HTILE_PIXELS_PER_TILE_TALL` はプラットフォームによって異なる（通常 8×8）。  
> `DepthExport` のスレッドグループサイズをこれに合わせることで、1グループ = 1 HTile タイルとなり、  
> Wave 集計（`WaveActiveMin/Max`）が正確にタイル境界に対応する。

> [!note]- DepthDecode が必要なケース
> `DepthDecode` は通常の SceneDepth が既に `DepthExport` で書き込まれているため、通常レンダリングでは使わない。  
> Virtual Shadow Map のページキャッシュ更新や `r.Nanite.DepthDecode` デバッグ CVar が有効な場合に限り、  
> HTile から深度の圧縮構造（ZMask, TileMinMax, TileIndex）をデバッグ用テクスチャに展開する目的で呼ばれる。
