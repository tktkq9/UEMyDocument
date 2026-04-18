# GPU a Ref: LightSampling シェーダーリファレンス

- シェーダー: `MegaLights/MegaLightsSampling.usf`, `MegaLightsRayTracing.usf`
- CPU 対応: [[b_megalights_sampling]]
- 上位: [[01_megalights_gpu_overview]]

---

## エントリポイント一覧

### MegaLightsSampling.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `GenerateLightSamplesCS` | 304 | CS | 確率的ライトサンプリング |
| `ClearLightSamplesCS` | 721 | CS | LightSamples バッファクリア |

### MegaLightsRayTracing.usf

| 関数名 | 行番号 | タイプ | 説明 |
|--------|--------|--------|------|
| `InitCompactedTraceTexelIndirectArgsCS` | 28 | CS | Indirect 引数初期化 |
| `CompactLightSampleTracesCS` | 83 | CS | 追跡対象サンプルのコンパクト化 |
| `ScreenSpaceRayTraceLightSamplesCS` | 226 | CS | SS レイトレース（HZB ベース）|
| `SoftwareRayTraceLightSamplesCS` | 351 | CS | GDF ソフトウェアレイトレース |
| `PrintTraceStatsCS` | 436 | CS | デバッグ統計出力 |

---

## GenerateLightSamplesCS パラメータ（MegaLightsSampling.usf）

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `DownsampledTileData` | `Buffer<uint>` | タイル座標パック（TILE_TYPE 別）|
| `DownsampledTileAllocator` | `Buffer<uint>` | タイル数カウンタ |
| `PackedPixelDataTexture` | `Texture2D<FMegaLightsPackedPixelData>` | 前フレームのピクセルデータ |
| `EncodedHistoryScreenCoordTexture` | `Texture2D<uint>` | 前フレームスクリーン座標（リプロジェクション）|
| `VisibleLightHashHistory` | `Texture2D` | 前フレームの可視ライットハッシュ |
| `BlueNoise` | `FBlueNoise` | Blue Noise テクスチャ |
| `DownsampledSceneDepth` | `Texture2D<float>` | ダウンサンプル済み深度 |
| `DownsampledSceneWorldNormal` | `Texture2D<UNORM float3>` | ダウンサンプル済みワールド法線 |
| `NumSamplesPerPixel` | `FIntPoint` | ピクセルあたりサンプル数 |
| `DownsampleFactor` | `FIntPoint` | ダウンサンプル係数（2×2 等）|
| `MinSampleWeight` | `float` | 最小サンプル重み |
| `MaxShadingWeight` | `float` | ファイアフライ抑制クランプ |

**出力:**

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `RWLightSamples` | `RWTexture2DArray` | サンプルライット情報（Index/Weight/Visibility）|
| `RWLightSampleRays` | `RWTexture2DArray` | シャドウレイ（Origin/Direction/TMax）|

---

## パーミュテーション（GenerateLightSamplesCS）

| マクロ | 値 | 説明 |
|--------|-----|------|
| `TILE_TYPE` | 0–12 | `TILE_MODE_SIMPLE_SHADING` 〜 `TILE_MODE_COMPLEX_SPECIAL_SHADING_RECT_TEXTURED` |
| `NUM_SAMPLES_PER_PIXEL_1D` | 1/2/4/16 | ピクセルあたりサンプル数 |
| `GUIDE_BY_HISTORY` | 0/1/2 | History PDF 補正の強度 |
| `DEBUG_MODE` | 0/1 | デバッグ表示 |

---

## TILE_MODE 定数（MegaLightsDefinitions.h）

| 定数 | 値 | 説明 |
|------|-----|------|
| `TILE_MODE_SIMPLE_SHADING` | 0 | 簡易シェーディング（Substrate FastPath）|
| `TILE_MODE_COMPLEX_SHADING` | 1 | 複合シェーディング |
| `TILE_MODE_EMPTY` | 6 | タイルなし（デフォルト）|
| `TILE_MODE_SINGLE_SHADING` | 7 | 単一シェーディング（Substrate SinglePath）|
| `TILE_MODE_COMPLEX_SPECIAL_SHADING` | 8 | 特殊複合シェーディング |
| `TILE_MODE_MAX` | 13 | モード数上限 |

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `MegaLightsSampling.ush` | FLightSampler, SampleLight(), SampleDirectionalLight() |
| `MegaLightsRayTracing.ush` | レイトレース共通型 |
| `BlueNoise.ush` | BlueNoise サンプリング関数 |
| `StochasticLighting/StochasticLightingCommon.ush` | MIS / RIS 共通関数 |
