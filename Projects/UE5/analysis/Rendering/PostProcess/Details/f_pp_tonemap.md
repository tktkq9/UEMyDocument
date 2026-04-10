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
