# REF: PostProcessMotionBlur.h / .cpp

- 対象ファイル: `Private/PostProcess/PostProcessMotionBlur.h` / `.cpp`
- 関連Details: [[e_pp_dof_motionblur]]

---

## enum

### `EMotionBlurQuality`

```cpp
enum class EMotionBlurQuality : uint8
{
    Low      = 0,  // 4〜8 サンプル
    Medium   = 1,  // 12〜16 サンプル
    High     = 2,  // 24 サンプル
    VeryHigh = 3,  // 32 サンプル（最高品質）
};
```

### `EMotionBlurFilter`

```cpp
enum class EMotionBlurFilter : uint8
{
    Unified,    // 単一パス（速度に沿ってサンプル収集）
    Separable,  // 水平・垂直分離パス（高品質エッジ）
};
```

---

## 構造体

### `FMotionBlurInputs`

```cpp
struct FMotionBlurInputs
{
    FScreenPassTexture SceneColor;         // 入力 SceneColor
    FScreenPassTexture SceneDepth;         // 深度バッファ
    FRDGTextureRef SceneVelocity;          // ベロシティバッファ
    EMotionBlurQuality Quality;            // 品質設定
    EMotionBlurFilter Filter;              // フィルター方式
    bool bOverrideVelocityBeforeDenoising; // デノイズ前の速度上書き
};
```

### `FMotionBlurOutputs`

```cpp
struct FMotionBlurOutputs
{
    FScreenPassTexture FullRes;                   // フル解像度出力
    FScreenPassTexture HalfRes;                   // ハーフ解像度出力
    FScreenPassTexture QuarterRes;                // クォーター解像度出力
    FRDGTextureRef VelocityFlattenTexture;        // タイル化速度テクスチャ
    FRDGTextureRef VelocityFlattenLookupTexture;  // 速度タイルルックアップ
};
```

---

## 主要関数

```cpp
// モーションブラーパスを追加
FMotionBlurOutputs AddMotionBlurPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FMotionBlurInputs& Inputs);

// モーションブラーが有効か
bool IsMotionBlurEnabled(const FViewInfo& View);

// モーションブラーの品質レベルを取得
EMotionBlurQuality GetMotionBlurQuality();

// ビジュアライズ（速度ベクトル可視化）→ VisualizeMotionVectors.cpp に委譲
FScreenPassTexture AddVisualizeMotionVectorsPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture SceneColor,
    FRDGTextureRef SceneVelocity);
```

---

## 内部パス構成

```
[1] VelocityFlattenCS
    → 画面を 16×16 タイルに分割
    → タイルごとの最大速度を記録
    → VelocityFlattenTexture に書き込み

[2] VelocityDilateCS
    → タイル間で速度を拡散（近傍タイルの最大速度に更新）
    → オブジェクト境界のアーティファクト低減

[3] MotionBlurCS / MotionBlurPS
    → 各ピクセルで速度方向に複数サンプルを収集
    → Unified: 1パスで完結
    → Separable: H パス → V パスの 2 段処理
    → サンプル数は Quality に応じて 4〜32

[4] （オプション）MotionBlurVisualizePS
    → 速度の方向・大きさを色分け可視化
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.MotionBlurQuality` | 4 | 品質（0=無効, 1=Low, 2=Medium, 3=High, 4=VeryHigh） |
| `r.MotionBlur.Filter` | 2 | フィルター方式（0=Separable, 1=Unified, 2=自動） |
| `r.MotionBlur.Amount` | 1 | ブラー強度スケール |
| `r.MotionBlur.Max` | 0 | 最大ブラー量（0=無制限） |
| `r.MotionBlur.TargetFPS` | 0 | ターゲット FPS（シャッタースピード正規化）|
| `r.MotionBlur.Scale` | 1.0 | スケール係数 |
| `r.MotionBlur.SeparableFilter.Distance` | 1.0 | Separable フィルター距離 |
| `r.VelocityOutputPass` | 0 | ベロシティ出力パス設定 |
| `r.BasePassOutputsVelocity` | 0 | BasePass でベロシティ出力 |
