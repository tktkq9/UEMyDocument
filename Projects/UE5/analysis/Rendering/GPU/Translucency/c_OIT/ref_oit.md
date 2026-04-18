# GPU c Ref: OIT シェーダーリファレンス

- シェーダー: `OIT/OITSorting.usf`, `OITCombine.usf`, `OITCommon.ush`
- CPU 対応: [[c_oit]]
- 上位: [[01_translucency_gpu_overview]]

---

## シェーダーファイル一覧

| ファイル | エントリポイント | 役割 |
|---------|---------------|------|
| `OIT/OITSorting.usf` | `MainCS`（複数パーミュテーション）| サンプルの深度ソート |
| `OITCombine.usf` | `MainCS`（`SHADER_OIT_COMBINE`）| ソート済みサンプルの合成 |
| `OITCombine.usf` | `MainCS`（`SHADER_OIT_SIMPLE_COMBINE`）| 簡易合成（低コスト）|

---

## OITSorting.usf パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `SampleDataTexture` | `Texture2DArray<uint>` | 入出力: サンプルデータ（色・深度をパック）|
| `SampleCountTexture` | `Texture2D<uint>` | 各ピクセルのサンプル数 |
| `Resolution` | `int2` | 出力バッファ解像度 |
| `MaxSideSampleCount` | `uint` | 片面最大サンプル数 |
| `MaxSampleCount` | `uint` | 合計最大サンプル数 |
| `Method` | `uint` | ソートアルゴリズム選択 |
| `PassType` | `uint` | `OIT_PASS_REGULAR` / `OIT_PASS_SEPARATE` |
| `SupportedPass` | `uint` | サポートするパスビットマスク |

---

## OITCombine.usf パラメータ

| HLSL 変数名 | 型 | 説明 |
|-----------|-----|------|
| `SampleDataTexture` | `Texture2DArray<uint>` | ソート済みサンプルデータ |
| `SampleCountTexture` | `Texture2D<uint>` | 各ピクセルのサンプル数 |
| `Resolution` | `int2` | バッファ解像度 |
| `MaxSampleCount` | `uint` | 最大サンプル数（= MAX_SAMPLE_COUNT）|

出力: `OutColor`（RWTexture2D）

---

## パーミュテーション

### OITSorting.usf

| マクロ | 値 | 説明 |
|--------|-----|------|
| `SORTING_SLICE_COUNT` | 1–8 | 同時処理スライス数（コンパイル時定数）|
| `BACK_TO_FRONT` | 0/1 | 背面→前面ソート |
| `FRONT_TO_BACK` | 0/1 | 前面→背面ソート |
| `ENCODING_TYPE` | 1 | `FPackedSliceAndOffset = uint` |
| `PRIMITIVE_TRILIST` | 0/1 | トライアングルリスト |
| `PRIMITIVE_TRISTRIP` | 0/1 | トライアングルストリップ |
| `PERMUTATION_DEBUG` | 0/1 | デバッグ ShaderPrint 有効 |
| `SHADER_DEBUG` | 0/1 | デバッグ出力 |

### OITCombine.usf

| マクロ | 値 | 説明 |
|--------|-----|------|
| `SHADER_OIT_COMBINE` | defined | 通常合成パス |
| `SHADER_OIT_SIMPLE_COMBINE` | defined | 簡易合成パス |

---

## OITCommon.ush 定数・型

```hlsl
#define OIT_ENABLED 1
#define MAX_SAMPLE_COUNT 8    // 最大サンプル数（コンパイル時定数）
#define OIT_PASS_REGULAR 1    // 通常の半透明パス
#define OIT_PASS_SEPARATE 2   // SeparateTranslucency パス

// サンプルデータのパック型
#define FPackedSliceAndOffset uint  // ENCODING_TYPE == 1 の場合
```

---

## サンプルバッファ仕様

| 項目 | 値 |
|------|----|
| テクスチャ型 | `Texture2DArray<uint>`（R32_UINT）|
| スライス数 | `MAX_SAMPLE_COUNT`（8）|
| 解像度 | スクリーン解像度と同一 |
| VRAM コスト | 解像度 × 8スライス × 4バイト = 1920×1080×8×4 ≈ 63 MB |

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.OIT.Mode` | 0: 無効, 1: 有効 |
| `r.OIT.MaxSampleCount` | 最大サンプル数（デフォルト 8、最大 8）|
| `r.OIT.SortType` | ソート方向 |
| `r.OIT.Debug` | デバッグ可視化 |
