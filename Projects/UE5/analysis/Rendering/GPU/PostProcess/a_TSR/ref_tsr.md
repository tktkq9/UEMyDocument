# リファレンス：TemporalSuperResolution.cpp エントリポイント

- グループ: a - TSR
- 上位: [[detail_tsr]]
- ソース: `Engine/Source/Runtime/Renderer/Private/PostProcess/TemporalSuperResolution.cpp`

## 概要

TSR の全 GPU パスを RDG 上に構築する実装ファイルのリファレンス。

---

## 主要関数

### `AddTemporalSuperResolutionPasses`

TSR の全パス（Reproject → UpdateHistory → Quantize）を RDG に追加するメインエントリ。

```cpp
// PostProcessing.cpp から呼び出される
ITemporalUpscaler::FOutputs AddTemporalSuperResolutionPasses(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const ITemporalUpscaler::FPassInputs& PassInputs);
```

内部で以下の RDG パスを発行:
1. `TSRClearPrevSceneColor` — ヒストリ初回クリア
2. `TSRReprojectHistory` — 前フレームヒストリの再投影
3. `TSRUpdateHistory` — 新フレームブレンド（品質パーミュテーションあり）
4. `TSROutputQuantization` — 出力解像度へのダウンスケール（200% モード時）

### `IsTemporalSuperResolutionEnabled`

```cpp
bool IsTemporalSuperResolutionEnabled(const FViewInfo& View);
```

`r.TSR.*` CVar と `EMainTAAPassConfig::TSR` の組み合わせで有効判定。

---

## 主要シェーダー（.usf）

| シェーダーファイル | パス | 説明 |
|----------------|------|------|
| `TSRReprojectHistory.usf` | ReprojectHistory | 前フレームヒストリ再投影 |
| `TSRUpdateHistory.usf` | UpdateHistory | ヒストリブレンド（DIM_UPDATE_QUALITY 0〜3） |
| `TSRClearPrevSceneColor.usf` | ClearPrevSceneColor | 初期化クリア |
| `TSROutputQuantization.usf` | OutputQuantization | 出力ダウンスケール |
| `TSRVisualizeHistory.usf` | VisualizeHistory | デバッグ可視化 |

---

## `EMainTAAPassConfig`（TemporalAA.h）

```cpp
enum class EMainTAAPassConfig : uint8
{
    Disabled,    // TAA 無効
    TAA,         // 旧来の TAA（Gen4 コンソール向け）
    TSR,         // Temporal Super Resolution（デフォルト）
    ThirdParty,  // サードパーティアップスケーラー
};
```

---

## 主要 CVar（再掲）

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.TSR.History.ScreenPercentage` | 100 | ヒストリ解像度倍率 |
| `r.TSR.History.SampleCount` | 16.0 | 最大サンプル数 |
| `r.TSR.History.UpdateQuality` | 3 | UpdateHistory 品質パーミュテーション |
| `r.TSR.WaveOps` | 1 | Wave 命令最適化 |
