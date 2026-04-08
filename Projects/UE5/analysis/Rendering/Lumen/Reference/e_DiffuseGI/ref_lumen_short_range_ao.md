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
    // Screen Probe 統合時に AO を適用するか
    bool ShouldApplyDuringIntegration();

    // スカラー AO の代わりに Bent Normal（3D ベクトル）を使うか
    // → Bent Normal は鏡面反射オクルージョンも改善（約 10% コスト増）
    bool UseBentNormal();

    // AO / Bent Normal テクスチャのピクセルフォーマットを返す
    EPixelFormat GetTextureFormat();

    // ダウンサンプル係数（2 = 半解像度で計算）
    uint32 GetDownsampleFactor();
    uint32 GetRequestedDownsampleFactor();

    // テンポラルフィルタを使うか
    bool UseTemporal();

    // テンポラルの隣接クランプスケール
    float GetTemporalNeighborhoodClampScale();

    // フォリッジ・サブサーフェスのオクルージョン強度上限
    float GetFoliageOcclusionStrength();
}
```

---

## FLumenScreenSpaceBentNormalParameters

Screen Space Bent Normal の参照パラメータ（LumenTracingUtils.h で定義）。  
Screen Probe 統合シェーダーがこのパラメータで AO テクスチャを参照する。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenScreenSpaceBentNormalParameters, )
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<UNORM float>, ShortRangeAOTexture)  // スカラー AO
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D<float3>,      ShortRangeGITexture)  // Bent Normal GI
    SHADER_PARAMETER(FIntPoint, ShortRangeAOViewMin)
    SHADER_PARAMETER(FIntPoint, ShortRangeAOViewSize)
    SHADER_PARAMETER(uint32,    ShortRangeAOMode)  // AO モード（0=スカラー, 1=BentNormal）
END_SHADER_PARAMETER_STRUCT()
```

---

## LumenScreenSpaceBentNormal.cpp

スクリーンスペーストレースで **Bent Normal を計算**するファイル。  
Depth Buffer + 法線から複数方向のレイをトレースし、遮蔽率を積算する。

```
計算手順:
  1. 各ピクセルから半球 N 方向に Short Range レイを発射（スクリーンスペース）
  2. Depth Buffer でヒット判定（SlopeCompareToleranceScale でスロープ閾値制御）
  3. ヒット方向を除外した平均方向 → Bent Normal
  4. 全方向のヒット率 → スカラー AO
  5. ダウンサンプル後テンポラル蓄積 → ノイズ低減
```

---

## LumenShortRangeAOHardwareRayTracing.cpp

Short Range AO の **HW RT** 実装。  
近距離のオクルージョンをより正確に計算するために RT を使う。

```cpp
// HW RT Short Range AO エントリポイント（LumenScreenProbeGather.h で宣言）
void RenderHardwareRayTracingShortRangeAO(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    ...); // FScreenProbeParameters と FLumenCardTracingParameters を受け取る
```

---

## 主要 CVar（LumenScreenSpaceBentNormal.cpp）

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
  │   ShortRangeGITexture（Bent Normal）を使って
  │   Irradiance の方向性を補正 → 遮蔽方向から来る照明を減らす
  │   → 鏡面反射オクルージョンにも影響
  │
  └─ UseBentNormal() = false:
      ShortRangeAOTexture（スカラー値 0〜1）を
      Irradiance に乗算 → 単純なオクルージョン
```
