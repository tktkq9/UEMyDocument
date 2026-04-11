# PostProcess b: TAA / TSR（テンポラルアンチエイリアス）

- 対象: `PostProcess/TemporalAA.h/.cpp` / `TemporalSuperResolution.cpp`
- 上位: [[05_postprocess_overview]]

---

## 役割

時間軸方向のサンプル蓄積によりアンチエイリアシングを行う2つのシステム。

| | TAA | TSR |
|--|-----|-----|
| クラス/実装 | `TemporalAA.h/.cpp` | `TemporalSuperResolution.cpp` |
| アップスケール | なし（native解像度） | あり（1/2〜1/4 → native） |
| ゴースト対策 | 速度ベース反発 | 履歴矩形クリッピング |
| シャープネス | 普通 | 高い |
| デフォルト | UE5以前・軽量ビルド | UE5 デフォルト |

---

## TAA（TemporalAA）

### EMainTAAPassConfig（メインAAモードの設定）

```cpp
enum class EMainTAAPassConfig : uint8
{
    Disabled,       // AA なし（FXAA等の後段処理に依存）
    TAA,            // 従来型 TAA
    TSR,            // TemporalSuperResolution（UE5標準）
    ThirdParty,     // DLSS / XeSS 等の外部プラグイン
};
```

### ETAAPassConfig（処理シナリオ別設定）

```cpp
enum class ETAAPassConfig : uint8
{
    Main,               // メインAA（SceneColor）
    MainUpsampling,     // アップサンプリング付きメインAA
    MainSuperSampling,  // スーパーサンプリング
    DiaphragmDOF,       // DOF ボケのためのTAA
    DiaphragmDOFUpsampling,
    LightShaft,
    ScreenSpaceAO,
    BackgroundSegmentation,
    HairStrands,
};
```

### FTAAPassParameters

```cpp
struct FTAAPassParameters
{
    ETAAPassConfig Pass;
    bool bIsComputePass;           // Compute シェーダーで実行するか

    // 入力テクスチャ
    FRDGTextureRef SceneColorInput;
    FRDGTextureRef SceneDepthTexture;
    FRDGTextureRef SceneVelocityTexture;

    // TAA 固有入力
    FRDGTextureRef PrevHistory;    // 前フレームの履歴バッファ
    FIntRect InputViewRect;
    FIntRect OutputViewRect;

    bool IsValid() const;          // パラメータ有効性チェック
    FIntPoint GetOutputExtent() const;
};
```

### 主要関数

```cpp
// TAA パスを追加（メインまたはサブパス）
FTAAOutputs AddTemporalAAPass(
    FRDGBuilder& GraphBuilder,
    const FSceneTextureParameters& SceneTextures,
    const FViewInfo& View,
    const FTAAPassParameters& Inputs,
    const FTAAOutputs* PreviousOutputs = nullptr);

// AA モードの判定
EMainTAAPassConfig GetMainTAAPassConfig(const FViewInfo& View);

// TSR が使われているか
bool IsTemporalAccumulationBasedMethod(EAntiAliasingMethod Method);
```

---

## TSR（TemporalSuperResolution）

### ITemporalUpscaler インターフェース

```cpp
// TSR/DLSS/XeSS 等の共通インターフェース（プラグイン差し替え対応）
class ITemporalUpscaler
{
public:
    struct FPassInputs
    {
        FRDGTextureRef SceneColorTexture;
        FRDGTextureRef SceneDepthTexture;
        FRDGTextureRef SceneVelocityTexture;
        FRDGTextureRef EyeAdaptationTexture;
        bool bAllowFullResInput;
        bool bGenerateOutputMip1;
        float SequenceNumber;
    };

    struct FOutputs
    {
        FRDGTextureRef FullRes;     // フル解像度出力
        FRDGTextureRef HalfRes;     // ハーフ解像度出力（オプション）
        FRDGTextureRef Mip1;        // MIP1（次フレームの TSR 履歴用）
    };

    // パスを追加する純粋仮想関数
    virtual FOutputs AddPasses(
        FRDGBuilder& GraphBuilder,
        const FViewInfo& View,
        const FPassInputs& PassInputs) const = 0;

    // このアップスケーラーの名前
    virtual const TCHAR* GetDebugName() const = 0;
};
```

### TSR の内部パス構成（TemporalSuperResolution.cpp）

```
[1] TSRClearPrevTexturesPass   … 前フレーム履歴の初期化（新規ビュー時）
[2] TSRDilateVelocityPass       … ベロシティバッファの拡張（近傍最大速度）
[3] TSRDecimateHistoryPass      … 解像度を下げた履歴バッファの作成
[4] TSRCompareTranslucencyPass  … 半透明のゴースト検出
[5] TSRRejectShadingPass        … シェーディング変化によるゴースト除去
    （履歴クリッピング + 鮮明度スコア計算）
[6] TSRSpatialAntiAliasingPass  … 空間的なエッジ処理
[7] TSRAccumulatePass           … 現フレームと履歴を混合（メインパス）
[8] TSRPostfilterHistoryPass    … 履歴フィルタリング
[9] TSRUpdateGuardBandPass      … ガードバンド更新
```

### TSR 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.TSR.History.ScreenPercentage` | 200 | 履歴バッファの解像度（%）|
| `r.TSR.History.SampleCount` | 16 | 履歴サンプル数 |
| `r.TSR.History.UpdateSpeed` | 3 | 履歴更新速度 |
| `r.TSR.Velocity.WeightClampingSampleCount` | 4 | 速度ベースウェイトクランプ |
| `r.TSR.ShadingRejection.Flickering` | 1 | フリッカリング除去 |
| `r.TSR.AsyncCompute` | 0 | 非同期コンピュート使用 |

---

## TAA 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.TemporalAA.Quality` | 2 | 品質（0=Low, 1=Medium, 2=High） |
| `r.TemporalAA.Algorithm` | 0 | アルゴリズム（0=Gen4, 1=Gen5） |
| `r.TemporalAA.Upsampling` | 0 | アップサンプリング有効 |
| `r.TemporalAA.FilterSize` | 1.0 | フィルターサイズ |
| `r.TemporalAA.Sharpen` | 0 | シャープネス強度 |

---

## コード実行フロー

### TAA エントリポイント

```
AddPostProcessingPasses()                      PostProcessing.cpp:347
  └─ AddTemporalAAPass()                       TemporalAA.cpp:571
       ├─ 入力: SceneColor / SceneDepth / SceneVelocity / 前フレーム履歴
       ├─ FTAAPassParameters で設定値を構築
       ├─ CS or VS/PS で Temporal Blend を実行
       └─ 出力: FTAAOutputs (SceneColor + SceneMetadata)
```

### TSR エントリポイント

```
AddPostProcessingPasses()
  └─ ITemporalUpscaler::AddPasses()
       └─ FTSRHistory（前フレーム履歴）を受け取り TSR 多パス実行
            ├─ DecimateHistory CS (履歴縮小)
            ├─ FilterAntiAlias CS (サブピクセル AA)
            ├─ RejectShading CS (履歴矩形クリッピング)
            └─ Upsample CS (解像度アップスケール)
```

### フロー詳細

1. **TAA** (`TemporalAA.cpp:571`) — `FTAAPassParameters` を構築し、`ETAAPassConfig` に応じた CS または VS/PS パスを選択する。前フレームの `FTAAOutputs::SceneMetadata` が履歴として渡される
2. **TSR** — `ITemporalUpscaler` インターフェース経由で呼ばれ、`FTSRHistory` に前フレームの高解像度履歴を保持する。デフォルト解像度の 50% 入力から native 解像度に復元する
3. ゴースト対策として TSR は「矩形クリッピング」（RejectShading）で履歴の誤サンプルを除去する
4. `GetMainTAAPassConfig()` が現在の AA 設定と解像度スケールから適切な `ETAAPassConfig` を選択する

### 関与クラス・関数一覧

| クラス/関数 | ファイル:行 | 役割 |
|------------|-----------|------|
| `AddTemporalAAPass()` | `TemporalAA.cpp:571` | TAA パスの実行 |
| `GetMainTAAPassConfig()` | `TemporalAA.h` | AA 設定の解決 |
| `ITemporalUpscaler::AddPasses()` | `TemporalAA.h` | TSR/DLSS 等の統一インターフェース |
| `FTAAPassParameters` | `TemporalAA.h` | TAA 入力パラメータ |
| `FTAAOutputs` | `TemporalAA.h` | TAA 出力（SceneColor + 履歴）|

---

## 関連リファレンス

- [[ref_pp_taa]] — FTAAPassParameters / AddTemporalAAPass リファレンス
- [[ref_pp_tsr]] — TSR / ITemporalUpscaler リファレンス
- [[ref_pp_orchestrator]] — パイプライン全体の実行順序
