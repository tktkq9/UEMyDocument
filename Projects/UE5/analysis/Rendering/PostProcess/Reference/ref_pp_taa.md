# REF: TemporalAA.h / .cpp

- 対象ファイル: `Private/PostProcess/TemporalAA.h` / `.cpp`
- 関連Details: [[b_pp_taa_tsr]]

---

## enum

### `EMainTAAPassConfig`

メインAA処理モードの設定。

```cpp
enum class EMainTAAPassConfig : uint8
{
    Disabled,    // AA なし
    TAA,         // 従来型 Temporal AA
    TSR,         // Temporal Super Resolution（UE5標準）
    ThirdParty,  // DLSS / XeSS 等の外部プラグイン
};
```

### `ETAAPassConfig`

TAA が処理するシナリオ（SceneColor 以外にも DOF・Hair 等で使用）。

```cpp
enum class ETAAPassConfig : uint8
{
    Main,                    // メイン SceneColor の AA
    MainUpsampling,          // アップサンプリング付きメイン AA
    MainSuperSampling,       // スーパーサンプリング
    DiaphragmDOF,            // DOF の Scatter/Gather に使用する TAA
    DiaphragmDOFUpsampling,  // DOF アップサンプリング
    LightShaft,              // ライトシャフト
    ScreenSpaceAO,           // SSAO のデノイズ
    BackgroundSegmentation,  // 背景分離
    HairStrands,             // ヘアストランドのデノイズ
};
```

### `ETAAQuality`

```cpp
enum class ETAAQuality : uint8
{
    Low    = 0,
    Medium = 1,
    High   = 2,
};
```

---

## 構造体

### `FTAAPassParameters`

TAA パスの入力パラメータ。

```cpp
struct FTAAPassParameters
{
    ETAAPassConfig Pass;            // 処理シナリオ
    bool bIsComputePass;            // Compute シェーダー使用

    // 入力テクスチャ
    FScreenPassTexture SceneColorInput;     // 現フレームのシーンカラー
    FRDGTextureRef SceneDepthTexture;       // 深度バッファ
    FRDGTextureRef SceneVelocityTexture;    // ベロシティバッファ

    // ビューポート設定
    FIntRect InputViewRect;                 // 入力ビューポート
    FIntRect OutputViewRect;                // 出力ビューポート（アップサンプリング時は異なる）

    // 解像度設定
    FIntPoint GetInputExtent() const;
    FIntPoint GetOutputExtent() const;

    bool IsValid() const;
};
```

### `FTAAOutputs`

TAA パスの出力データ。

```cpp
struct FTAAOutputs
{
    FScreenPassTexture SceneColor;      // AA 済み SceneColor
    FScreenPassTexture SceneMetadata;   // メタデータ（次フレームの履歴）
    FScreenPassTexture DownsampledSceneColor; // ダウンサンプル済みシーンカラー
};
```

---

## 主要関数

```cpp
// TAA パスを追加
FTAAOutputs AddTemporalAAPass(
    FRDGBuilder& GraphBuilder,
    const FSceneTextureParameters& SceneTextures,
    const FViewInfo& View,
    const FTAAPassParameters& Inputs,
    const FTAAOutputs* PreviousOutputs = nullptr);
// PreviousOutputs: 前フレームの TAA 出力（null = 初回フレーム）

// メインAA処理モードを決定
EMainTAAPassConfig GetMainTAAPassConfig(const FViewInfo& View);

// テンポラル蓄積ベースのAA手法か判定
bool IsTemporalAccumulationBasedMethod(EAntiAliasingMethod Method);

// TAA が DOF パスで使用されるか
bool IsTAAUpsamplingConfig(ETAAPassConfig Pass);

// タイル分割で処理するか
bool IsLowResTemporalAAConfig(ETAAPassConfig Pass);
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.TemporalAA.Quality` | 2 | TAA 品質（0=Low, 1=Medium, 2=High） |
| `r.TemporalAA.Algorithm` | 0 | TAA アルゴリズム世代（0=Gen4, 1=Gen5） |
| `r.TemporalAA.Upsampling` | 0 | TAA アップサンプリング有効 |
| `r.TemporalAA.FilterSize` | 1.0 | 時間フィルターサイズ |
| `r.TemporalAA.Sharpen` | 0 | シャープネス強度 |
| `r.TemporalAA.CatmullRom.Sharpness` | 0.9 | Catmull-Rom シャープネス |
| `r.TemporalAA.ReprojectionBlendFactor` | 0.9 | 再投影ブレンド係数 |
| `r.TemporalAA.VelocityWeighting` | 1 | 速度重み付け |
| `r.TemporalAA.ForceComputeShader` | 0 | Compute シェーダー強制 |
| `r.TemporalAA.HistoryScreenPercentage` | 100 | 履歴バッファ解像度（%）|

---

> [!note]- ETAAPassConfig とシナリオ別の使い分け
> `ETAAPassConfig` は単なる「TAA ON/OFF」ではなく、DOF・ダウンサンプル等のシナリオも含む列挙型。  
> `Disabled` は AA を完全にスキップ、`TAA` は通常の native 解像度 AA、  
> `DiaphragmDOF_*` は被写界深度パス内で独自の Temporal Accumulation を使うモード。  
> `GetMainTAAPassConfig()` が ViewInfo の設定から適切な値を自動選択する。

> [!note]- FTAAOutputs::SceneMetadata と履歴の管理
> `FTAAOutputs::SceneMetadata` は次フレームの TAA に渡される「テンポラル履歴」テクスチャ。  
> `PreviousOutputs` として渡された前フレームの `SceneMetadata` と現フレームの SceneColor を  
> 速度ベースのブレンド係数でブレンドする。初回フレーム（`PreviousOutputs = nullptr`）は履歴なしで実行される。

> [!note]- IsTemporalAccumulationBasedMethod によるサブシステムの判別
> Lumen や VSM など多くのサブシステムが「テンポラル蓄積 AA が有効か」で動作を変える。  
> `IsTemporalAccumulationBasedMethod(EAntiAliasingMethod)` が TAA / TSR の場合に true を返し、  
> これを見て Lumen が Temporal Filtering を有効にしたり VSM がページキャッシュを有効にしたりする。
