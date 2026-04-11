# REF: Tonemap・LUT ファイル群

- 対象ファイル:
  - `PostProcessTonemap.h/.cpp`
  - `PostProcessCombineLUTs.h/.cpp`
  - `ACESUtils.h/.cpp`
- 関連Details: [[f_pp_tonemap]]

---

## PostProcessTonemap.h/.cpp

### FTonemapInputs

```cpp
struct FTonemapInputs
{
    // 入力テクスチャ
    FScreenPassTexture SceneColor;         // HDR SceneColor
    FScreenPassTexture Bloom;              // Bloom テクスチャ（加算合成）
    FRDGBufferRef EyeAdaptationBuffer;     // 露出値（float1 バッファ）
    FRDGTextureRef ColorGradingTexture;    // 合成済み 3D LUT（CombineLUTs 出力）
    FRDGTextureRef LocalExposureTexture;   // ローカル露出補正テクスチャ（nullptr=無効）
    FRDGTextureRef BlurredLogLuminanceTexture; // ローカル露出用 log 輝度

    // 設定フラグ
    bool bWriteAlphaChannel = false;       // アルファチャンネル書き込み
    bool bOutputInHDR = false;             // HDR ディスプレイ向け出力
    bool bMetalMSAAHDRDecode = false;      // Metal MSAA HDR デコード
    bool bGammaOnly = false;               // ガンマ変換のみ（トーンマップをスキップ）
    bool bFlipYAxis = false;               // Y 軸反転（モバイル対応）

    // 出力設定
    FScreenPassRenderTarget Output;        // 出力先レンダーターゲット
};
```

### FTonemapperOutputDeviceParameters

```cpp
// 出力デバイスのカラー設定（シェーダーバインド用）
BEGIN_SHADER_PARAMETER_STRUCT(FTonemapperOutputDeviceParameters, )
    SHADER_PARAMETER(FVector3f, InverseGamma)       // 逆ガンマ係数
    SHADER_PARAMETER(uint32,    OutputDevice)        // 出力デバイス種別
    SHADER_PARAMETER(uint32,    OutputGamut)         // 出力色域
    SHADER_PARAMETER(float,     OutputMaxLuminance)  // 最大輝度（HDR向け）
END_SHADER_PARAMETER_STRUCT()
```

### 主要関数

```cpp
// Tonemap パスを追加
FScreenPassTexture AddTonemapPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FTonemapInputs& Inputs);

// 出力デバイスパラメータを取得（View Family 設定から）
FTonemapperOutputDeviceParameters GetTonemapperOutputDeviceParameters(
    const FSceneViewFamily& Family);

// モバイル向け Tonemap パス
FScreenPassTexture AddMobileTonemapperPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FTonemapInputs& Inputs);
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Tonemapper.Quality` | 5 | トーンマッパー品質（0〜5）|
| `r.Tonemapper.GrainQuantization` | 1 | フィルムグレイン量子化 |
| `r.Tonemapper.Sharpen` | 0 | シャープネス強度 |
| `r.Tonemapper.ACESInversion` | 1 | ACES 逆変換精度 |
| `r.Tonemapper.StochasticGrainVolumetric` | 0 | ボリューメトリックグレイン |
| `r.HDR.Display.OutputDevice` | 0 | HDR 出力デバイス |
| `r.HDR.Display.ColorGamut` | 0 | HDR カラーガマット |
| `r.HDR.UI.CompositeMode` | 1 | HDR UI 合成モード |
| `r.Color.UseColorSpacePassThrough` | 0 | カラースペース素通し |

---

## PostProcessCombineLUTs.h/.cpp

### 役割

複数のカラーグレーディング LUT を1つの 32×32×32 3D テクスチャに合成する。  
合成結果は Tonemap シェーダーが参照する。

### 構造体

```cpp
// カラーリマップパラメータ（シェーダーバインド用）
BEGIN_SHADER_PARAMETER_STRUCT(FColorRemapParameters, )
    SHADER_PARAMETER(FVector3f, MappingPolynomial)  // 輝度マッピング多項式（A, B, C）
END_SHADER_PARAMETER_STRUCT()
// MappingPolynomial: Y = A * X^2 + B * X + C の係数
```

### 主要関数

```cpp
// LUT 合成パスを追加（結果は FRDGTextureRef として返す）
FRDGTextureRef AddCombineLUTPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View);
// 出力: 32×32×32 RGBA8 Texture3D（線形空間）
// 入力: PostProcessSettings の ColorGrading テクスチャ群（最大16枚をブレンド）
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.LUT.Size` | 32 | LUT サイズ（32³）|
| `r.LUT.Texture.FullDomain` | 0 | LUT のフルドメイン範囲 |

---

## ACESUtils.h/.cpp

### 役割

ACES（Academy Color Encoding System）2.0 の色域変換に使用するルックアップテーブルを  
GPU バッファとして管理する。Tonemap シェーダーが参照する。

### 構造体

```cpp
// Tonemap シェーダーへのバインドパラメータ
BEGIN_SHADER_PARAMETER_STRUCT(FACESTonemapShaderParameters, )
    SHADER_PARAMETER_SRV(Buffer<float4>, ReachMTable)        // Reach-M テーブル SRV
    SHADER_PARAMETER_SRV(Buffer<float4>, GamutCuspTable)     // 色域境界テーブル SRV
    SHADER_PARAMETER_SRV(Buffer<float4>, UpperHullGammaTable)// 上側ハルガンマテーブル SRV
END_SHADER_PARAMETER_STRUCT()
```

### 主要関数

```cpp
// ACES 変換テーブルリソースを取得（初回は GPU バッファを生成してキャッシュ）
FACESTonemapShaderParameters GetTransformResources(FRHICommandListBase& RHICmdList);
```

### テーブルの内容

| テーブル | 役割 |
|---------|------|
| `ReachMTable` | ACES 2.0 の Reach-M（最大輝度マッピング）係数 |
| `GamutCuspTable` | 各色相の色域境界点（クラスプ）座標 |
| `UpperHullGammaTable` | 上側凸包ガンマカーブ補間テーブル |

---

> [!note]- CombineLUT の 32³ 3D テクスチャとブレンド
> `AddCombineLUTPass()` (`PostProcessCombineLUTs.cpp:494`) は PostProcessSettings の ColorGrading 設定を  
> 32×32×32 の 3D テクスチャ（RGBA8, 線形空間）に焼き込む CS パス。  
> 最大 16 枚の外部 LUT テクスチャを重み付きブレンドで合成できる。  
> Tonemap シェーダーはこの 1 枚の 3D テクスチャだけをサンプリングするため、複数グレーディングでも実行コストは固定���

> [!note]- ACES 2.0 と GetTransformResources の事前生成テーブル
> `ACESUtils::GetTransformResources()` はプロセス起動後初回呼び出し時に  
> `ReachMTable`・`GamutCuspTable`・`UpperHullGammaTable` を CPU で計算して GPU バッファにキャッシュする。  
> ACES 2.0 は gamut mapping を動的計算するのではなく、これらのルックアップテーブルを使って  
> シェーダー内の計算を高速化してい���。テーブルは以降のフレームでも再利用される。

> [!note]- AddTonemapPass の HDR モニター出力
> `AddTonemapPass()` (`PostProcessTonemap.cpp:569`) は `r.HDR.Display.OutputDevice` によって  
> sRGB 出力（SDR）か PQ / HLG（HDR10）かを切り替え���。  
> HDR 出力の場合、ガンマ変換の代わりに SMPTE ST 2084 の PQ カーブを適用して  
> HDR モニターが扱える 0〜1 の信号値にマッピングする（最大輝度 10000 nit を想定）。
