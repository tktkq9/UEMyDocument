# GPU c Ref: Composite シェーダーリファレンス

- シェーダー: `MegaLights/MegaLights.usf`, `MegaLightsVolumeSampling.usf`, `MegaLightsVolumeRayTracing.usf`, `MegaLightsVolumeShading.usf`
- CPU 対応: [[a_megalights_pipeline]], [[d_megalights_resolve]]
- 上位: [[01_megalights_gpu_overview]]

---

## エントリポイント一覧

### MegaLights.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `MegaLightsTileClassificationBuildListsCS` | 70 | CS | Substrate タイル分類・リスト構築 |
| `InitTileIndirectArgsCS` | 136 | CS | Indirect Dispatch 引数初期化 |
| `HairTransmittanceCS` | 188 | CS | Hair Strands トランスミッタンス計算 |

### MegaLightsVolumeSampling.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `VolumeGenerateLightSamplesCS` | 161 | CS | フロクセル単位ライトサンプリング |

### MegaLightsVolumeRayTracing.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `VolumeCompactLightSampleTracesCS` | 36 | CS | ボリュームトレースのコンパクト化 |
| `VolumeSoftwareRayTraceLightSamplesCS` | 140 | CS | GDF ベースソフトウェアレイトレース |

### MegaLightsVolumeShading.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `VolumeShadeLightSamplesCS` | 99 | CS | フロクセル照明評価 → SH ボリューム書き込み |

---

## MegaLightsTileClassificationBuildListsCS パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `MegaLightsTileBitmask` | `Texture2D<uint>` | Substrate タイルビットマスク |
| `RWTileAllocator` | `RWBuffer<uint>` | タイルタイプ別アロケーター |
| `RWTileData` | `RWBuffer<uint>` | タイル座標バッファ |
| `RWDownsampledTileAllocator` | `RWBuffer<uint>` | ダウンサンプルタイルアロケーター |
| `RWDownsampledTileData` | `RWBuffer<uint>` | ダウンサンプルタイル座標バッファ |
| `ViewMinInTiles` | `uint2` | ビューのタイル空間最小座標 |
| `ViewSizeInTiles` | `uint2` | ビューのタイル空間サイズ |
| `DOWNSAMPLE_FACTOR_X` / `Y` | マクロ | ダウンサンプル係数 |

---

## VolumeShadeLightSamplesCS パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `VolumeLightSamples` | `Texture3D` | フロクセル毎のサンプルデータ |
| `RWTranslucencyAmbient` | `RWTexture3D<float4>` | 出力: アンビエント SH（カスケード別）|
| `RWTranslucencyDirectional` | `RWTexture3D<float4>` | 出力: ディレクショナル SH（カスケード別）|
| `VolumePhaseG` | `float` | Henyey-Greenstein 非対称パラメータ |

---

## TILE_MODE 定数

| 定数 | 値 | Substrate パス |
|------|-----|--------------|
| `TILE_MODE_SIMPLE_SHADING` | 0 | FastPath |
| `TILE_MODE_COMPLEX_SHADING` | 1 | 複合 |
| `TILE_MODE_SIMPLE_SHADING_RECT` | 2 | FastPath + RectLight |
| `TILE_MODE_COMPLEX_SHADING_RECT` | 3 | 複合 + RectLight |
| `TILE_MODE_SIMPLE_SHADING_RECT_TEXTURED` | 4 | FastPath + Textured RectLight |
| `TILE_MODE_COMPLEX_SHADING_RECT_TEXTURED` | 5 | 複合 + Textured RectLight |
| `TILE_MODE_EMPTY` | 6 | 空タイル |
| `TILE_MODE_SINGLE_SHADING` | 7 | SinglePath |
| `TILE_MODE_COMPLEX_SPECIAL_SHADING` | 8 | 特殊複合 |
| `TILE_MODE_MAX` | 13 | 上限 |

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `MegaLights.ush` | FMegaLightsData, IsEnabled(), UseVolume() |
| `MegaLightsTileClassification.ush` | GetTileModeFromBitmask(), BuildTileMode(), MEGALIGHTS_TILE_BITMASK_* |
| `MegaLightsVolume.ush` | フロクセル座標変換関数 |
| `/Engine/Shared/MegaLightsDefinitions.h` | `TILE_MODE_*` 定数定義 |
