# PostProcess f: Tonemap・LUT・ACES

- 対象: `PostProcess/PostProcessTonemap.h/.cpp` / `PostProcessCombineLUTs.h/.cpp` / `ACESUtils.h/.cpp`
- 上位: [[05_postprocess_overview]]

---

## 役割

HDR SceneColor を**表示可能な LDR に変換**する最終カラーパイプライン。

```
HDR SceneColor
    ↓
CombineLUTs（Color LUT の合成）
    ↓
Tonemap（露出適用 + ACES / ユーザー設定 + LUT サンプリング）
    ↓
LDR 最終出力
```

---

## 1. PostProcessCombineLUTs（LUT合成）

### 役割

複数のカラーグレーディング LUT（ルックアップテーブル）を1枚の 3D テクスチャに合成する。  
Tonemap シェーダーはこの合成済み LUT だけを参照するため、複数グレーディングでも低コスト。

```cpp
// LUT 合成パスを追加（結果は 32×32×32 の 3D LUT テクスチャ）
FRDGTextureRef AddCombineLUTPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View);

// カラーリマップパラメータ（シェーダーバインド用）
BEGIN_SHADER_PARAMETER_STRUCT(FColorRemapParameters, )
    SHADER_PARAMETER(FVector3f, MappingPolynomial)  // 輝度マッピング多項式係数
END_SHADER_PARAMETER_STRUCT()
```

### LUT の使用

- インプット: UE のカラーグレーディングアセット（最大16枚の LUT をブレンド）
- アウトプット: 32×32×32 の RGBA Texture3D
- カラーグレーディング・トーンカーブ・彩度・コントラスト等を合成

---

## 2. PostProcessTonemap（トーンマッパー）

### 役割

Tonemap の入力データをまとめた構造体 `FTonemapInputs` を受け取り、  
HDR → LDR 変換・カラーグレーディング LUT サンプリング・Bloom 合成を行う。

### FTonemapInputs

```cpp
struct FTonemapInputs
{
    // 入力テクスチャ
    FScreenPassTexture SceneColor;         // HDR SceneColor
    FScreenPassTexture Bloom;              // Bloom テクスチャ
    FRDGBufferRef EyeAdaptationBuffer;     // 露出値（EyeAdaptation 結果）
    FRDGTextureRef ColorGradingTexture;    // 合成済み LUT（CombineLUTs 出力）
    FRDGTextureRef LocalExposureTexture;   // ローカル露出テクスチャ（オプション）

    // 設定
    bool bWriteAlphaChannel;
    bool bOutputInHDR;                     // HDR ディスプレイ出力
    ETonemapperOutputDevice OutputDevice;  // ターゲットデバイス
};

// Tonemap パスを追加
FScreenPassTexture AddTonemapPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FTonemapInputs& Inputs);
```

### 出力デバイス設定

```cpp
// トーンマッパーの出力デバイス（色域・ガンマ）
FTonemapperOutputDeviceParameters GetTonemapperOutputDeviceParameters(
    const FSceneViewFamily& Family);

// 出力デバイスの種類（内部enum）
// sRGB / Rec709 / ExplicitGammaMapping / ACES1000nitST2084 / ACES2000nit 等
```

### Tonemap の処理内容

```
1. EyeAdaptation から露出値を取得
2. LocalExposure テクスチャをサンプリング（有効な場合）
3. Bloom テクスチャを SceneColor に加算合成
4. Neutral Tonemap または ACES トーンカーブを適用
5. カラーグレーディング LUT でサンプリング（色補正）
6. 出力デバイスに合わせた色域変換（sRGB / HDR10 / Dolby Vision 等）
7. Grain（フィルムノイズ）の追加（オプション）
8. Vignette（周辺減光）の適用（オプション）
```

---

## 3. ACESUtils（ACES 2.0 色空間変換）

### 役割

ACES（Academy Color Encoding System）2.0 の色域変換ルックアップテーブルを GPU にアップロードし、  
Tonemap シェーダーが ACES カラースペースで正確な変換を行えるようにする。

```cpp
// ACES 変換テーブルのリソース
struct FACESTonemapShaderParameters
{
    FShaderResourceViewRHIRef ReachMTableSRV;       // Reach-M テーブル
    FShaderResourceViewRHIRef GamutCuspTableSRV;    // 色域境界テーブル
    FShaderResourceViewRHIRef UpperHullGammaTableSRV; // 上側ハルガンマテーブル
};

// テーブルリソースを取得（初回呼び出し時に GPU バッファ生成）
FACESTonemapShaderParameters GetTransformResources(FRHICommandListBase& RHICmdList);
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Tonemapper.Quality` | 5 | トーンマッパー品質（0〜5） |
| `r.Tonemapper.GrainQuantization` | 1 | グレイン量子化 |
| `r.Tonemapper.Sharpen` | 0 | シャープネス強度 |
| `r.Tonemapper.ACESInversion` | 1 | ACES 逆変換（ACES→リニアの正確性） |
| `r.LUT.Size` | 32 | LUT サイズ（32³ デフォルト） |
| `r.Color.UseColorSpacePassThrough` | 0 | カラースペース素通し |
| `r.Bloom.EnableCross` | 0 | 十字形 Bloom をトーンマップ前に合成 |
| `r.HDR.Display.OutputDevice` | 0 | HDR 出力デバイス |
| `r.HDR.Display.ColorGamut` | 0 | HDR カラーガマット |
| `r.HDR.UI.CompositeMode` | 1 | HDR UI 合成モード |

---

## コード実行フロー

### エントリポイント

```
AddPostProcessingPasses()
  │
  ├─ AddCombineLUTPass()               PostProcessCombineLUTs.cpp:494
  │    ├─ PostProcessSettings の ColorGrading 設定を読み込む
  │    ├─ 最大 16 枚の Color Grading LUT をブレンド重みで合成
  │    └─ 出力: 32³ RGBA8 3D テクスチャ（CombinedLUT）
  │
  └─ AddTonemapPass()                  PostProcessTonemap.cpp:569
       ├─ 入力: SceneColor (HDR) + EyeAdaptationBuffer + CombinedLUT
       ├─ 露出を EV 値で適用
       ├─ ACES トーンマッピングカーブ適用（ACESUtils.cpp で事前生成テーブル参照）
       ├─ CombinedLUT でカラーグレーディング適用
       └─ 出力: LDR SceneColor（ガンマ適用済み）
```

### フロー詳細

1. **AddCombineLUTPass()** (`PostProcessCombineLUTs.cpp:494`) — `FPostProcessSettings` の Color Grading 設定から Saturation / Contrast / Gamma / Gain / Offset を取り出し、32³ の 3D LUT を CS で生成する。複数のグレーディング LUT は重み付き平均でブレンドされる
2. **AddTonemapPass()** (`PostProcessTonemap.cpp:569`) — EyeAdaptation バッファから現フレームの EV を読み取り、ACES 2.0 カーブを使って HDR → LDR に変換する。最後に CombinedLUT をサンプリングしてカラーグレーディングを適用する
3. `r.Tonemapper.Quality` が高いほど LUT サイズが大きくなり精度が向上する

### 関与クラス・関数一覧

| クラス/関数 | ファイル:行 | 役割 |
|------------|-----------|------|
| `AddCombineLUTPass()` | `PostProcessCombineLUTs.cpp:494` | Color Grading LUT 合成 |
| `AddTonemapPass()` | `PostProcessTonemap.cpp:569` | ACES トーンマップ + LUT 適用 |
| `GetTransformResources()` | `ACESUtils.h` | ACES 変換テーブル取得 |

---

## 関連リファレンス

- [[ref_pp_tonemap]] — AddTonemapPass / AddCombineLUTPass / ACESUtils リファレンス
- [[ref_pp_exposure]] — EyeAdaptation による露出値の供給
- [[ref_pp_orchestrator]] — パイプライン全体の実行順序
