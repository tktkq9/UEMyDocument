# REF: 被写界深度ファイル群

- 対象ファイル:
  - `DiaphragmDOF.h/.cpp` / `DiaphragmDOFUtils.cpp`
  - `PostProcessBokehDOF.h/.cpp`
  - `PostProcessDOF.h/.cpp`
- 関連Details: [[e_pp_dof_motionblur]]

---

## DiaphragmDOF.h/.cpp + DiaphragmDOFUtils.cpp

物理ベースのカメラ絞りをモデル化した高品質 DOF 実装。

### FPhysicalCocModel（物理ベースCoC計算モデル）

```cpp
struct FPhysicalCocModel
{
    // カメラパラメータ
    float FocusDistance;           // フォーカス距離（cm）
    float LensF;                   // 焦点距離 (mm)
    float FStop;                   // F値（絞り）
    float Aperture;                // 開口（FStop から計算）

    // CoC 半径の上限（画面幅比）
    float MaxForegroundRadius;     // 前景ボケの最大半径
    float MaxBackgroundRadius;     // 背景ボケの最大半径
    float InfinityBackgroundRadius;// 無限遠背景の CoC 半径

    // Petzval 収差（周辺ボケの変形）
    float PetzvalRatio;            // 収差比率（0=なし, 1=最大）

    // CoC 半径を計算
    float ComputeForegroundCocRadius(float SceneDepth) const;
    float ComputeBackgroundCocRadius(float SceneDepth) const;

    // センサーサイズから f 値を計算
    static FPhysicalCocModel FromSettings(const FSceneView& View);
};
```

### FBokehModel（ボケ形状モデル）

```cpp
enum class EBokehShape : uint8
{
    Circle,          // 円形（最速・GPU コスト最小）
    StraightBlades,  // 直線羽根（多角形ボケ）
    RoundedBlades,   // 丸み付き羽根
};

struct FBokehModel
{
    EBokehShape BokehShape;
    float DiaphragmBladeCount;     // 羽根枚数（4〜16）
    float DiaphragmBladeRoundness; // 丸み（0=直線, 1=円形）
    float DiaphragmBladeCurvature; // 曲率
    float DiaphragmRotation;       // 回転角度（度）

    // カスタムボケテクスチャ（nullptr = プリセット形状を使用）
    const FTextureResource* BokehShapeTexture;

    // ボケ形状からパーミュテーションを決定
    static FBokehModel FromSettings(const FSceneView& View);
};
```

### DiaphragmDOF 名前空間

```cpp
namespace DiaphragmDOF
{
    // DOF が有効かどうか
    bool IsEnabled(const FViewInfo& View);

    // DOF パス一式を追加
    FRDGTextureRef AddPasses(
        FRDGBuilder& GraphBuilder,
        const FSceneTextureParameters& SceneTextures,
        const FViewInfo& View,
        FRDGTextureRef InputSceneColor,
        const FTranslucencyPassResources& TranslucencyViewResources);

    // CoC モデルの最大半径を取得（Compute シェーダー用）
    float GetMaxBackgroundRadius(const FViewInfo& View, float ViewportWidth);
}
```

### DiaphragmDOF 内部パス

```
[1] DiaphragmDOFReduceCS      … CoC マップ生成 + 前景/背景分割ダウンサンプル
[2] DiaphragmDOFGatherCS      … ボケ Gather（CoC 内サンプル収集）
    ├── 前景 Gather パス
    └── 背景 Gather パス
[3] DiaphragmDOFScatterOccCS  … 前景→背景のオクルージョン計算（オプション）
[4] DiaphragmDOFPostFilterCS  … ボケ後フィルタリング
[5] DiaphragmDOFRecombineCS   … 前景 + 背景 + シャープ層の合成
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.DepthOfFieldQuality` | 2 | DOF 品質（0=無効, 4=最高） |
| `r.DOF.Kernel.MaxForegroundRadius` | 0.025 | 前景最大 CoC 半径（画面幅比） |
| `r.DOF.Kernel.MaxBackgroundRadius` | 0.025 | 背景最大 CoC 半径 |
| `r.DOF.Gather.AccumulatorQuality` | 1 | Gather アキュムレータ品質（0/1） |
| `r.DOF.Gather.EnableBokehSettings` | 1 | ボケ形状設定を使用 |
| `r.DOF.Scatter.MainPassPerObjectRatio` | 0.1 | Scatter メインパスのオブジェクト割合 |
| `r.DOF.Scatter.ForegroundCompositing` | 1 | 前景ボケ合成 |
| `r.DOF.Scatter.BackgroundCompositing` | 2 | 背景ボケ合成 |
| `r.DOF.TemporalAAQuality` | 1 | DOF 用 TAA 品質 |
| `r.DOF.HalfRes` | 1 | ハーフ解像度処理 |

---

## PostProcessBokehDOF.h/.cpp

### 役割

DOF の**可視化デバッグパス**。CoC マップや焦点領域を色分け表示する。  
（実際のボケ描画は DiaphragmDOF が担当）

### 構造体・関数

```cpp
struct FVisualizeDOFInputs
{
    FScreenPassTexture SceneColor;
    FScreenPassTexture SceneDepth;
};

FScreenPassTexture AddVisualizeDOFPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FVisualizeDOFInputs& Inputs);
// CoC マップを色分け表示（前景=緑, 焦点=白, 背景=赤）
```

---

## PostProcessDOF.h/.cpp

### 役割

**旧来の簡易 DOF 実装**（Shader Model 4 以下向け・レガシー）。  
現在は DiaphragmDOF が標準で、モバイル等の軽量パスとして残存する。

```cpp
// 旧 DOF パスを追加（非推奨・モバイル向け）
FScreenPassTexture AddDepthOfFieldPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    FRDGTextureRef SceneDepth);
```
