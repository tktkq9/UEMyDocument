# GPU c: OIT — Order-Independent Translucency

- シェーダー: `OIT/OITSorting.usf`, `OITCombine.usf`
- CPU 対応: [[c_oit]]
- 上位: [[01_translucency_gpu_overview]]

---

## 概要

**OIT（Order-Independent Translucency）** は深度ソートなしで半透明を正確に合成する手法。  
UE5 の OIT は **サンプルリスト方式（Per-Pixel Linked List 的手法）** を採用しており、  
各ピクセルに最大 N サンプルを収集してから GPU 上でソート・合成する。

通常の半透明（Painter's Algorithm）と異なり、オブジェクトの描画順序を問わず  
正しい前後関係で合成できる。

---

## 処理フロー

```
【収集フェーズ】
OIT 対応の半透明マテリアルを描画
  → 各フラグメントのデータ（色・深度・サンプルインデックス）を
    SampleDataTexture（Texture2DArray<uint>）に書き込み
  → SampleCountTexture（Texture2D<uint>）で各ピクセルのサンプル数をカウント

【ソートフェーズ】
OITSorting.usf の MainCS（複数パーミュテーション）
  → 各ピクセルの最大 MAX_SAMPLE_COUNT サンプルを深度でソート
     - BACK_TO_FRONT: 背面から前面へ（Additive ブレンド向け）
     - FRONT_TO_BACK: 前面から背面へ（Transmittance 積算向け）

【合成フェーズ】
OITCombine.usf の MainCS（SHADER_OIT_COMBINE / SHADER_OIT_SIMPLE_COMBINE）
  → ソート済みサンプルを順番にアルファブレンドして最終色を計算
```

---

## OITSorting.usf — ソート CS

`OIT/OITSorting.usf`

```hlsl
#define MAX_SAMPLE_COUNT 8  // 最大サンプル数（ピクセルあたり）
#define OIT_ENABLED 1

// パーミュテーション: SORTING_SLICE_COUNT, PERMUTATION_DEBUG, PRIMITIVE_TRILIST/TRISTRIP

[numthreads(THREADGROUP_SIZE, 1, 1)]
void MainCS(uint3 DispatchThreadId : SV_DispatchThreadID, ...)
{
    // 1. SampleDataTexture から最大 MAX_SAMPLE_COUNT サンプルを読み出し
    // 2. 深度でバブルソート（小規模なのでレジスタソートが効率的）
    // 3. ソート済みサンプルを書き戻し
}
```

**パーミュテーション変数:**

| マクロ | 値 | 説明 |
|--------|-----|------|
| `SORTING_SLICE_COUNT` | 1–8 | ソート対象スライス数 |
| `BACK_TO_FRONT` | 0/1 | 背面→前面ソート |
| `FRONT_TO_BACK` | 0/1 | 前面→背面ソート |
| `ENCODING_TYPE` | 1 | パックエンコーディング種別 |
| `PRIMITIVE_TRILIST` | 0/1 | トライアングルリスト入力 |
| `PRIMITIVE_TRISTRIP` | 0/1 | トライアングルストリップ入力 |

---

## OITCombine.usf — 合成 CS

`OITCombine.usf`

```hlsl
// SHADER_OIT_COMBINE: 通常合成（深度ソート済みサンプルを前後ブレンド）
[numthreads(THREADGROUP_SIZE, 1, 1)]
void MainCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    uint2 PixelCoord = DispatchThreadId.xy;
    uint SampleCount = SampleCountTexture[PixelCoord];

    float4 AccumColor = float4(0, 0, 0, 1); // アルファ = 透過率

    for (uint i = 0; i < min(SampleCount, MaxSampleCount); i++)
    {
        // SampleDataTexture[PixelCoord, i] からサンプルデータをデコード
        float4 SampleColor = DecodeSample(SampleDataTexture[uint3(PixelCoord, i)]);
        // アルファブレンド（前→後の順に蓄積）
        AccumColor.rgb += SampleColor.rgb * AccumColor.a;
        AccumColor.a   *= (1.0 - SampleColor.a);
    }

    // 最終色を出力
    OutColor[PixelCoord] = float4(AccumColor.rgb, 1.0 - AccumColor.a);
}
```

---

## サンプルデータエンコーディング

```hlsl
// OITCommon.ush

// FPackedSliceAndOffset: depth/color をコンパクトにパック（ENCODING_TYPE == 1 で uint）
#define FPackedSliceAndOffset uint

// 各サンプルエントリは SampleDataTexture (Texture2DArray<uint>) に格納
// Slice = サンプルインデックス, uint = エンコード済み色・深度
```

---

## 適用条件と通常半透明との使い分け

| 条件 | 使用するシステム |
|------|--------------|
| `r.OIT.Mode = 0` | 通常半透明（Painter's Algorithm）|
| `r.OIT.Mode = 1` | OIT（深度ソート不要・正確な合成）|
| 交差する半透明メッシュ | OIT が必要（Painter's では破綻）|
| 粒子・スプライト | 通常半透明で十分なことが多い |

---

## MaxSideSampleCount

```hlsl
uint MaxSideSampleCount; // 片面あたりの最大サンプル数
uint MaxSampleCount;     // 合計最大サンプル数 (= MAX_SAMPLE_COUNT)
```

`r.OIT.MaxSampleCount` で制御。増やすと精度向上・VRAM 消費増。

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `r.OIT.Mode` | 0: 無効, 1: OIT 有効 |
| `r.OIT.MaxSampleCount` | 最大サンプル数（デフォルト 8）|
| `r.OIT.SortType` | ソート方向（BackToFront / FrontToBack）|
