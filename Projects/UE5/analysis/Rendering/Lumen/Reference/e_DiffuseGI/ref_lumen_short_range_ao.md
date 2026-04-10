# リファレンス：LumenShortRangeAO.h / LumenShortRangeAOHardwareRayTracing.cpp / LumenScreenSpaceBentNormal.cpp

- グループ: e - Diffuse GI
- 上位: [[e_lumen_diffuse_gi]]
- 関連: [[ref_lumen_screen_probe_gather]] | [[ref_lumen_tracing_utils]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenShortRangeAO.h`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenShortRangeAOHardwareRayTracing.cpp`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenScreenSpaceBentNormal.cpp`

---

## 概要

Screen Probe Gather の**近距離 AO（Short Range AO）**を提供するファイル群。  
スクリーンスペースの小さなスケールの遮蔽情報を **Bent Normal**（平均非遮蔽方向）として計算し、  
Screen Probe Gather の統合時に適用してコンタクトシャドウや洞窟内の暗化を再現する。  
SW（スクリーンスペーストレース）と HW RT の 2 系統を持つ。

---

## LumenShortRangeAO 名前空間（LumenShortRangeAO.h）

```cpp
namespace LumenShortRangeAO {
    bool ShouldApplyDuringIntegration();
    bool UseBentNormal();
    EPixelFormat GetTextureFormat();
    uint32 GetDownsampleFactor();
    uint32 GetRequestedDownsampleFactor();
    bool UseTemporal();
    float GetTemporalNeighborhoodClampScale();
    float GetFoliageOcclusionStrength();
}
```

### 関数一覧

| 関数 | 戻り値 | 説明 |
|------|--------|------|
| `ShouldApplyDuringIntegration()` | `bool` | Screen Probe 統合時に AO を適用するか。`r.Lumen.ScreenProbeGather.ShortRangeAO` が有効な場合 |
| `UseBentNormal()` | `bool` | スカラー AO の代わりに Bent Normal（3D ベクトル）を使うか。`r.Lumen.ScreenProbeGather.ShortRangeAO.BentNormal` |
| `GetTextureFormat()` | `EPixelFormat` | AO テクスチャのピクセルフォーマット（UseBentNormal=true なら float3、false なら float）|
| `GetDownsampleFactor()` | `uint32` | 実際に使用するダウンサンプル係数（GetRequestedDownsampleFactor をクランプした値）|
| `GetRequestedDownsampleFactor()` | `uint32` | `r.Lumen.ScreenProbeGather.ShortRangeAO.DownsampleFactor` の値 |
| `UseTemporal()` | `bool` | テンポラルフィルタを使うか（`r.Lumen.ScreenProbeGather.ShortRangeAO.Temporal`）|
| `GetTemporalNeighborhoodClampScale()` | `float` | テンポラルの隣接クランプスケール（大きいほどゴースティング増・ノイズ減）|
| `GetFoliageOcclusionStrength()` | `float` | フォリッジ・サブサーフェスのオクルージョン強度上限（デフォルト 0.7）|

### 使用箇所

- [[ref_lumen_short_range_ao]] — `LumenScreenSpaceBentNormal.cpp` が `GetDownsampleFactor()` / `UseBentNormal()` 等で動作を決定
- [[ref_lumen_short_range_ao]] — `LumenShortRangeAOHardwareRayTracing.cpp` でも同様に参照
- [[ref_lumen_screen_probe_filtering]] — 統合パスが `ShouldApplyDuringIntegration()` / `UseBentNormal()` でシェーダー分岐を決定

---

## FLumenScreenSpaceBentNormalParameters

Screen Space Bent Normal の参照パラメータ（LumenTracingUtils.h で定義）。  
Screen Probe 統合シェーダーがこのパラメータで AO テクスチャを参照する。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenScreenSpaceBentNormalParameters, )
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<UNORM float>, ShortRangeAOTexture)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float3>,      ShortRangeGITexture)
    SHADER_PARAMETER(FIntPoint, ShortRangeAOViewMin)
    SHADER_PARAMETER(FIntPoint, ShortRangeAOViewSize)
    SHADER_PARAMETER(uint32,    ShortRangeAOMode)
END_SHADER_PARAMETER_STRUCT()
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `ShortRangeAOTexture` | `Texture2D<UNORM float>` | スカラー AO テクスチャ（0〜1。`UseBentNormal=false` の場合に使用）|
| `ShortRangeGITexture` | `Texture2D<float3>` | Bent Normal GI テクスチャ（方向ベクトル。`UseBentNormal=true` の場合に使用）|
| `ShortRangeAOViewMin` | `FIntPoint` | AO テクスチャのビュー領域左上座標 |
| `ShortRangeAOViewSize` | `FIntPoint` | AO テクスチャのビュー領域サイズ |
| `ShortRangeAOMode` | `uint32` | AO モード（0=スカラー AO, 1=Bent Normal）|

### 使用箇所

- [[ref_lumen_tracing_utils]] — `FLumenScreenSpaceBentNormalParameters` がヘッダに定義される
- [[ref_lumen_screen_probe_filtering]] — 統合パスで `ShortRangeAOMode` に応じて AO または Bent Normal を適用

---

## LumenScreenSpaceBentNormal.cpp

スクリーンスペーストレースで **Bent Normal を計算**するファイル。  
Depth Buffer + 法線から複数方向のレイをトレースし、遮蔽率を積算する。

### 計算手順

1. **スクリーン空間レイの生成**
   ```cpp
   // 各ピクセルから半球 N 方向（DownsampleFactor に応じてサンプル数調整）のレイを生成
   // FrameIndex に応じて方向をジッター → テンポラル蓄積でカバー率を向上
   ```

2. **Depth Buffer によるヒット判定**
   ```cpp
   // HZB を使用して各方向にスクリーン空間レイを発射
   // SlopeCompareToleranceScale でスロープ閾値を制御
   // → 値が小さいほど厳密（薄いサーフェスのセルフシャドウを避ける）
   ```

3. **Bent Normal の計算**
   ```cpp
   // 非遮蔽方向の平均 → Bent Normal ベクトル（ShortRangeGITexture に格納）
   // 全方向の遮蔽率の平均 → スカラー AO（ShortRangeAOTexture に格納）
   ```

4. **ダウンサンプル + テンポラル蓄積**
   ```cpp
   // GetDownsampleFactor() でダウンサンプル（デフォルト: 1/2 解像度）
   // UseTemporal() が true なら前フレームと NeighborhoodClampScale でブレンド
   ```

---

## LumenShortRangeAOHardwareRayTracing.cpp

Short Range AO の **HW RT** 実装。  
近距離のオクルージョンをより正確に計算するために RT を使う。

```cpp
void RenderHardwareRayTracingShortRangeAO(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FLumenCardTracingParameters& TracingParameters,
    const FScreenProbeParameters& ScreenProbeParameters,
    FLumenScreenSpaceBentNormalParameters& OutBentNormalParameters,
    ERDGPassFlags ComputePassFlags);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（RT-BLAS / TLAS アクセスに使用）|
| `View` | `const FViewInfo&` | カメラビュー |
| `TracingParameters` | `const FLumenCardTracingParameters&` | Surface Cache アトラス・トレース共通パラメータ |
| `ScreenProbeParameters` | `const FScreenProbeParameters&` | プローブ座標・解像度情報 |
| `OutBentNormalParameters` | `FLumenScreenSpaceBentNormalParameters&` | 出力: AO / Bent Normal テクスチャを格納 |
| `ComputePassFlags` | `ERDGPassFlags` | AsyncCompute / Compute の選択フラグ |

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — `r.Lumen.ScreenProbeGather.ShortRangeAO.HardwareRayTracing` が有効な場合に SW RT の代わりに呼ばれる

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.ScreenProbeGather.ShortRangeAO.DownsampleFactor` | 2 | 計算解像度ダウンサンプル係数 |
| `r.Lumen.ScreenProbeGather.ShortRangeAO.Temporal` | 1 | テンポラルフィルタの有効/無効 |
| `r.Lumen.ScreenProbeGather.ShortRangeAO.BentNormal` | 1 | スカラー AO の代わりに Bent Normal を使うか |
| `r.Lumen.ScreenProbeGather.ShortRangeAO.Temporal.NeighborhoodClampScale` | 1.0 | テンポラルクランプの許容度（大きいほどゴースティング増・ノイズ減）|
| `r.Lumen.ScreenProbeGather.ShortRangeAO.SlopeCompareToleranceScale` | 0.5 | スロープ閾値スケール（ヒット判定の厳密さ）|
| `r.Lumen.ScreenProbeGather.ShortRangeAO.FoliageOcclusionStrength` | 0.7 | フォリッジ/サブサーフェス用オクルージョン上限 |
| `r.Lumen.ScreenProbeGather.ShortRangeAO.MaxMultibounceAlbedo` | 0.5 | マルチバウンス AO 計算の最大アルベド |

---

## Short Range AO の統合

```
Screen Probe Gather 統合パス:
  ShouldApplyDuringIntegration() = true の場合:

  ├─ UseBentNormal() = true:
  │   ShortRangeGITexture（Bent Normal ベクトル）を使って
  │   Irradiance の方向性を補正 → 遮蔽方向から来る照明を減らす
  │   → 鏡面反射オクルージョンにも影響（ShortRangeAOMode = 1）
  │
  └─ UseBentNormal() = false:
      ShortRangeAOTexture（スカラー値 0〜1）を
      Irradiance に乗算 → 単純なオクルージョン（ShortRangeAOMode = 0）

フォリッジ・サブサーフェス:
  GetFoliageOcclusionStrength() でオクルージョン上限をクランプ
  → 葉などの Subsurface マテリアルが暗くなりすぎるのを防ぐ
```
