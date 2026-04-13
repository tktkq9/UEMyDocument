# リファレンス：PostProcessMotionBlur.h エントリポイント

- グループ: f - Motion Blur
- 上位: [[detail_motion_blur]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessMotionBlur.h/.cpp`

## 主要関数

| 関数 | 説明 |
|------|------|
| `AddMotionBlurPass(GraphBuilder, View, Inputs)` | Motion Blur RDG パスを追加し `FMotionBlurOutputs` を返す |
| `AddVelocityFlattenPass(GraphBuilder, View, SceneVelocity, SceneDepth)` | Velocity Flatten + Dilate パスを追加し `FVelocityFlattenTextures` を返す |
| `IsMotionBlurEnabled(View)` | Motion Blur が有効かどうかを判定 |
| `GetMotionBlurQuality()` | `r.MotionBlurQuality` CVar から `EMotionBlurQuality` を取得 |
| `GetMotionBlurFilter()` | `r.MotionBlur.Filter` CVar から `EMotionBlurFilter` を取得 |

---

## `EMotionBlurQuality` 列挙型

```cpp
enum class EMotionBlurQuality : uint32
{
    Low,      // 低品質（最低サンプル数）
    Medium,   // 中品質
    High,     // 高品質
    VeryHigh, // 最高品質
    MAX
};
```

## `EMotionBlurFilter` 列挙型

```cpp
enum class EMotionBlurFilter : uint32
{
    Unified,   // 統合フィルタ（推奨）
    Separable, // 縦横分離フィルタ
};
```

---

## `FVelocityFlattenTextures` 全メンバ

| メンバ | 型 | 説明 |
|--------|-----|------|
| `VelocityFlatten` | `FScreenPassTexture` | Tile 内最大ベロシティ縮小テクスチャ |
| `VelocityTileArray` | `FScreenPassTexture` | Dilate 済みベロシティ Tile 配列 |

---

## `FMotionBlurInputs` 全メンバ

| 変数名 | 型 | 必須 | 説明 |
|--------|-----|------|------|
| `SceneColor` | `FScreenPassTexture` | ✅ | HDR シーンカラー |
| `SceneDepth` | `FScreenPassTexture` | ✅ | シーン深度テクスチャ |
| `SceneVelocity` | `FScreenPassTexture` | ✅ | ピクセルベロシティバッファ |
| `VelocityFlattenTextures` | `FVelocityFlattenTextures` | ✅ | Tile 縮小・Dilate 済みベロシティ |
| `LensDistortionLUT` | `FRDGTextureRef` | ― | レンズ歪み補正 LUT（オプション） |
| `Quality` | `EMotionBlurQuality` | ✅ | ブラー品質 |
| `Filter` | `EMotionBlurFilter` | ✅ | フィルタ方式 |
| `VelocityScale` | `float` | ― | ベロシティスケール倍率 |

---

## `FMotionBlurOutputs` 全メンバ

| メンバ | 型 | 説明 |
|--------|-----|------|
| `MotionBlurColor` | `FScreenPassTexture` | Motion Blur 適用済みシーンカラー |

---

## シェーダーパーミュテーション

```cpp
// PostProcessMotionBlur.cpp 内の代表的なパーミュテーション次元
class FMotionBlurQualityDim : SHADER_PERMUTATION_ENUM_CLASS("MOTION_BLUR_QUALITY", EMotionBlurQuality);
class FMotionBlurFilterDim : SHADER_PERMUTATION_ENUM_CLASS("MOTION_BLUR_FILTER", EMotionBlurFilter);
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.MotionBlurQuality` | 4 | 0=無効, 1=Low, 2=Medium, 3=High, 4=VeryHigh |
| `r.MotionBlur.Amount` | 1.0 | ブラー強度スケール |
| `r.MotionBlur.Max` | 0 | 最大ベロシティ制限（0=無制限） |
| `r.MotionBlur.TargetFPS` | 0 | FPS 正規化（0=無効） |
| `r.MotionBlur.Filter` | 0 | 0=Unified, 1=Separable |
| `r.MotionBlur.SampleCount` | 0 | サンプル数オーバーライド |
