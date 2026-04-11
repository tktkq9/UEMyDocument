# PostProcess e: 被写界深度・モーションブラー

- 対象: `PostProcess/DiaphragmDOF.h/.cpp` / `PostProcessBokehDOF.h/.cpp` / `PostProcessMotionBlur.h/.cpp`
- 上位: [[05_postprocess_overview]]

---

## 1. DiaphragmDOF（高品質被写界深度）

### 役割

物理的なカメラ絞り（Diaphragm）をモデル化した高品質 DOF。  
CoC（Circle of Confusion）マップを生成し、前景・後景を分離してブラーをかける。

### 物理モデル

```cpp
// 物理ベースの CoC 計算モデル
struct FPhysicalCocModel
{
    float FocusDistance;        // フォーカス距離（cm）
    float MaxForegroundRadius;  // 前景 CoC 半径上限
    float MaxBackgroundRadius;  // 背景 CoC 半径上限
    float InfinityBackgroundRadius; // 無限遠背景の CoC 半径

    // Petzval 収差補正（周辺ボケの変形）
    float PetzvalRatio;

    // f値（FStop）から CoC を計算
    float ComputeForegroundCocRadius(float SceneDepth) const;
    float ComputeBackgroundCocRadius(float SceneDepth) const;
};
```

### ボケ形状モデル

```cpp
// ボケ（Bokeh）の形状設定
struct FBokehModel
{
    EBokehShape BokehShape;
    float DiaphragmBladeCount;      // 絞り羽根枚数（角形ボケに影響）
    float DiaphragmBladeRoundness;  // 羽根の丸み（0=直線, 1=円形）
    float DiaphragmBladeCurvature;  // 羽根の曲率
    float DiaphragmRotation;        // 絞り回転角度

    // カスタムボケテクスチャ（任意の形状）
    const FTextureResource* BokehShapeTexture;
};

enum class EBokehShape : uint8
{
    Circle,          // 円形（最速）
    StraightBlades,  // 直線羽根（多角形ボケ）
    RoundedBlades,   // 丸み付き羽根
};
```

### パイプライン（DiaphragmDOF::AddPasses）

```
[1] ReducePass（Gather/Scatter 前処理）
    → SceneColor + SceneDepth → CoC マップ生成
    → 前景/背景に分割したダウンサンプル

[2] GatherPass（Gather モード）
    → CoC 半径内のサンプルを収集してボケを生成
    → 前景ボケ（前からの滲み）と背景ボケを個別処理

[3] ScatterOcclusionPass（オプション）
    → 前景ボケが背景に重なる際のオクルージョン処理

[4] RecombinePass
    → 前景ボケ + 背景ボケ + シャープ層を合成
    → 最終 SceneColor に出力
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.DepthOfFieldQuality` | 2 | DOF 品質（0=無効, 4=最高） |
| `r.DOF.Kernel.MaxForegroundRadius` | 0.025 | 前景最大CoC半径（画面幅比） |
| `r.DOF.Kernel.MaxBackgroundRadius` | 0.025 | 背景最大CoC半径 |
| `r.DOF.Gather.AccumulatorQuality` | 1 | Gather アキュムレータ品質 |
| `r.DOF.Scatter.MainPassPerObjectRatio` | 0.1 | ScatterパスオブジェクトあたりのメインパスRatio |
| `r.DOF.TemporalAAQuality` | 1 | DOF 用TAA品質 |

---

## 2. PostProcessBokehDOF（DOF可視化）

### 役割

DiaphragmDOF の結果や DoF パラメータを**可視化デバッグ表示**するためのパス。  
実際のボケ描画ではなく、CoC マップや焦点領域を色分けして確認できる。

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
```

---

## 3. PostProcessMotionBlur（モーションブラー）

### 役割

カメラ移動・オブジェクト移動による**速度ベースのブラー**を生成する。  
ベロシティバッファを使ってピクセルの移動量を計算しブラーをかける。

### 品質・フィルター設定

```cpp
enum class EMotionBlurQuality : uint8
{
    Low      = 0, // 少ないサンプル数
    Medium   = 1,
    High     = 2,
    VeryHigh = 3, // 最高サンプル数
};

enum class EMotionBlurFilter : uint8
{
    Unified,    // 単一パス（速度）
    Separable,  // 分離パス（高品質）
};
```

### 出力構造体

```cpp
struct FMotionBlurOutputs
{
    FScreenPassTexture FullRes;     // フル解像度出力
    FScreenPassTexture HalfRes;     // ハーフ解像度（高速化）
    FScreenPassTexture QuarterRes;  // クォーター解像度
    FRDGTextureRef VelocityFlattenTexture; // 速度タイルテクスチャ
};
```

### パイプライン

```
[1] VelocityFlatten（速度タイル化）
    → 画面を 16×16 タイルに分割
    → 各タイルの最大速度を記録（Scatter 最適化用）

[2] VelocityDilate（速度拡張）
    → タイル間の速度を拡散してエッジのアーティファクトを低減

[3] MotionBlur 本体
    → 各ピクセルで速度方向にサンプルを収集
    → Unified: 単一パス / Separable: 水平・垂直分離
```

### 主要関数

```cpp
struct FMotionBlurInputs
{
    FScreenPassTexture SceneColor;
    FScreenPassTexture SceneDepth;
    FRDGTextureRef SceneVelocity;
    EMotionBlurQuality Quality;
    EMotionBlurFilter Filter;
};

FMotionBlurOutputs AddMotionBlurPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FMotionBlurInputs& Inputs);
```

### 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.MotionBlurQuality` | 4 | モーションブラー品質（0=無効, 4=最高） |
| `r.MotionBlur.Filter` | 2 | フィルター方式（0=Separable, 1=Unified, 2=自動） |
| `r.MotionBlur.Amount` | 1 | ブラー強度スケール |
| `r.MotionBlur.Max` | 0 | 最大ブラー量（0=制限なし） |
| `r.MotionBlur.TargetFPS` | 0 | ターゲットFPS（シャッタースピード正規化） |
| `r.MotionBlur.Scale` | 1.0 | スケール係数 |

---

## コード実行フロー

### DiaphragmDOF エントリポイント

```
AddPostProcessingPasses()
  └─ DiaphragmDOF::AddPasses()             DiaphragmDOF.cpp:1486
       ├─ SetupCS: CoC（Circle of Confusion）マップ生成
       ├─ ReduceCS: 前景・後景の分離 + ダウンサンプル
       ├─ GatherCS (Foreground): 前景ボケを収集（蓄積型）
       ├─ GatherCS (Background): 後景ボケを収集
       ├─ PostFilterCS: CoC 境界の修正・前景ボケの後景への合成
       └─ ScatterCS: 高輝度スポットを散乱（ボケシェイプ使用時）
```

### MotionBlur エントリポイント

```
AddPostProcessingPasses()
  └─ AddMotionBlurPass()                   PostProcessMotionBlur.cpp:1333
       ├─ MotionBlurSetup CS: ベロシティタイル最大値を計算
       ├─ MotionBlurFlatten CS: 隣接タイルのベロシティを統合
       ├─ MotionBlurDilate CS: タイルベロシティを拡張
       └─ MotionBlurApply CS: タイルごとにブラーを適用（Gather 方式）
```

### フ���ー詳細

1. **DiaphragmDOF** (`DiaphragmDOF.cpp:1486`) — `FPhysicalCocModel` でカメラの焦点距離・絞り・センサーサイズから CoC を物理計算する。前景と後景のボケは非対称なので分離処理する
2. **GatherCS** — CoC 半径内のピクセルを「収集（Gather）」してボケを生成する。ScatterCS は明るい点光源のボケシェイプ（六角形・円形等）を散乱で表現する
3. **MotionBlur** (`PostProcessMotionBlur.cpp:1333`) — タイル単位でベロシティの最大値を求め、カメラモーション（ビュー行列差分）とオブジェクトモーション（ベロシティバッファ）を合算してブラーをかける
4. `r.MotionBlurQuality` が 3 以上でオブジェクトモーションブラー有効、0 で全て無効

### 関与クラス・関数一覧

| クラス/関数 | ファイル:行 | 役割 |
|------------|-----------|------|
| `DiaphragmDOF::AddPasses()` | `DiaphragmDOF.cpp:1486` | 高品質 DOF 全パス |
| `FPhysicalCocModel` | `DiaphragmDOF.h` | 物理ベース CoC 計算 |
| `AddMotionBlurPass()` | `PostProcessMotionBlur.cpp:1333` | モーションブラー |

---

## 関連リファレンス

- [[ref_pp_dof]] — DiaphragmDOF / BokehDOF リファレンス
- [[ref_pp_motionblur]] — PostProcessMotionBlur リファレンス
