# リファレンス：TemporalAA.h / TemporalAA.cpp エントリポイント

- グループ: b - TAA
- 上位: [[detail_taa]]
- ソース: `Engine/Source/Runtime/Renderer/Private/PostProcess/TemporalAA.h/.cpp`

## 概要

TAA パスの主要インターフェースとシェーダー設定のリファレンス。

---

## 主要関数

### `AddTemporalAAPass`

```cpp
FTAAOutputs AddTemporalAAPass(
    FRDGBuilder& GraphBuilder,
    const FSceneTextureParameters& Inputs,
    const FViewInfo& View,
    const FTAAPassParameters& Params,
    const FTemporalAAHistory& InputHistory,
    FTemporalAAHistory* OutputHistory);
```

| 引数 | 説明 |
|------|------|
| `Params.Pass` | `ETAAPassConfig` でパス種別を選択 |
| `Params.Quality` | `ETAAQuality` で Low/Medium/High/MediumHigh を選択 |
| `InputHistory` | 前フレームのヒストリテクスチャ |
| `OutputHistory` | 今フレームのヒストリ（次フレームへの引き渡し用） |

---

## 列挙型

### `EMainTAAPassConfig`

```cpp
enum class EMainTAAPassConfig : uint8
{
    Disabled,    // AA 無効
    TAA,         // 旧来の UE4 TAA
    TSR,         // Temporal Super Resolution
    ThirdParty,  // サードパーティアップスケーラー
};
```

### `ETAAPassConfig`

```cpp
enum class ETAAPassConfig
{
    Main,                     // メインシーンカラー AA
    MainUpsampling,           // TAA + アップサンプリング
    MainSuperSampling,        // 高品質スーパーサンプリング
    ScreenSpaceReflections,   // SSR ノイズ蓄積
    LightShaft,               // Light Shaft ノイズ蓄積
    DiaphragmDOF,             // DOF CoC 込み TAA
    DiaphragmDOFUpsampling,   // DOF TAA + アップサンプリング
    Hair,                     // ヘア用 TAA
    MAX
};
```

### `ETAAQuality`

```cpp
enum class ETAAQuality : uint8
{
    Low, Medium, High, MediumHigh, MAX
};
```

---

## `FTAAPassParameters` メンバ

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `Pass` | `ETAAPassConfig` | 実行するパス種別 |
| `Quality` | `ETAAQuality` | 品質設定 |
| `bOutputRenderTargetable` | `bool` | RT として出力可能か |
| `bDownsample` | `bool` | 半解像度ダウンサンプルを同時出力するか |
| `InputViewRect` | `FIntRect` | 入力ビューポート矩形 |
| `OutputViewRect` | `FIntRect` | 出力ビューポート（アップサンプリング時は大きい） |
| `ResolutionDivisor` | `int32` | 解像度除数（通常 1） |
| `SceneDepthTexture` | `FRDGTexture*` | デプステクスチャ |
| `SceneVelocityTexture` | `FRDGTexture*` | ベロシティテクスチャ |
| `SceneColorInput` | `FRDGTexture*` | 入力シーンカラー |
| `SceneMetadataInput` | `FRDGTexture*` | DOF CoC 等のメタデータ |
| `CoCBilateralFilterStrength` | `float` | DOF TAA バイラテラルフィルタ強度 |

---

## ユーティリティ関数

| 関数 | 説明 |
|------|------|
| `IsTAAUpsamplingConfig(Pass)` | MainUpsampling / DiaphragmDOFUpsampling / MainSuperSampling かどうか |
| `IsMainTAAConfig(Pass)` | Main / MainUpsampling / MainSuperSampling かどうか |
| `IsDOFTAAConfig(Pass)` | DiaphragmDOF / DiaphragmDOFUpsampling かどうか |
| `FTAAPassParameters::SetupViewRect(View, Divisor)` | InputViewRect / OutputViewRect を View から設定 |
| `FTAAPassParameters::GetOutputExtent()` | 出力テクスチャ解像度を返す |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.TemporalAA.Algorithm` | 0 | 0=TAA, 1=TSR |
| `r.TemporalAA.Quality` | 3 | 品質レベル |
| `r.TemporalAA.Upsampling` | 0 | アップサンプリング有効化 |
| `r.TemporalAA.FilterSize` | 1.0 | フィルタサイズ |
| `r.TemporalAA.CatmullRom` | 1 | Catmull-Rom ヒストリサンプリング |
