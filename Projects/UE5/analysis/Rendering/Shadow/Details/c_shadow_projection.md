# c: Shadow Projection（PCF / PCSS）

- 対象ファイル: `ShadowRendering.cpp` / `ShadowProjectionPixelShader.h`
- 概要: [[17_shadow_overview]]

---

## 概要

Shadow Projection は **Shadow Depth Map との深度比較によってシャドウの有無を判定**し、  
PCF（Percentage Closer Filtering）または PCSS（Percentage Closer Soft Shadows）で  
柔らかいシャドウを生成して Shadow Mask テクスチャに書き込む。

---

## Shadow Projection の種類

| 手法 | CVar | 説明 |
|-----|------|------|
| PCF | `r.Shadow.FilterMethod=0` | 固定半径の近傍サンプル平均 |
| PCSS | `r.Shadow.FilterMethod=1` | 遮蔽物サイズに応じた可変半径（より自然なソフトシャドウ）|

---

## FShadowProjectionPS テンプレート

```cpp
// ShadowProjectionPixelShader.h
template<uint32 Quality, bool bUseFadePlane = false, bool bModulatedShadows = false>
class TShadowProjectionPS : public FShadowProjectionPixelShaderInterface
{
    // Quality: サンプル数（1/2/4/8/16/32）
    // 高 Quality ほどソフトシャドウが滑らかだが GPU コスト増大
    //
    // bUseFadePlane: カスケード間のフェードに使用
    // bModulatedShadows: モバイル向け乗算合成シャドウ

    void SetParameters(
        FRHIBatchedShaderParameters& BatchedParameters,
        int32 ViewIndex,
        const FSceneView& View,
        const FProjectedShadowInfo* ShadowInfo)
    {
        // Shadow Matrix（受光面 → Shadow UV への変換）
        // Shadow Depth Atlas テクスチャ
        // ShadowDepthBias（アクネ防止）
        // ShadowFadeFraction（カスケードフェード）
    }
};
```

---

## PCSS の仕組み

```
PCSS（Percentage Closer Soft Shadows）:

[1] 遮蔽物サイズ推定（BlockerSearch）
    Shadow Depth Atlas を r.Shadow.PCSS.BlockerSearchNumSamples 個でサンプル
    → 遮蔽物の平均深度 d_blocker を取得

[2] ペニンブラサイズ計算
    PenumbraRadius = (d_receiver - d_blocker) × LightSize / d_blocker
    → 遮蔽物が近いほど小さく（ハードシャドウ）
    → 遮蔽物が遠いほど大きく（ソフトシャドウ）

[3] PCF フィルタリング
    PenumbraRadius に応じた半径で PCF を実行
    → 近傍サンプルの影有無を平均 → ShadowFactor（0.0〜1.0）
```

---

## RenderProjectedShadow() フロー

```
RenderLights() 内、各 UnbatchedLight に対して:
FProjectedShadowInfo::RenderProjectedShadow()   ShadowRendering.cpp
  │
  ├─ [1] Shadow Mask テクスチャ確保
  │   ShadowMaskTexture = RDGBuilder.CreateTexture(R8, "ShadowMask")
  │
  ├─ [2] ステンシル設定（オプション）
  │   Point / Spot Light:
  │     球・コーン境界ジオメトリをステンシルに描画
  │     → 境界外のピクセルをスキップ（コスト削減）
  │   Directional Light:
  │     全画面クワッドのため ステンシルスキップ
  │
  ├─ [3] TShadowProjectionPS<Quality> 描画
  │   SetParameters() で Shadow パラメータをバインド:
  │     ShadowViewProjectionMatrix（受光面 → Shadow 空間）
  │     ShadowDepthTexture（Atlas）
  │     ShadowDepthSampler
  │     ShadowDepthBias（深度バイアス = r.Shadow.DepthBias）
  │
  ├─ [4] シェーダー内での深度比較
  │   float ShadowDepth = ShadowDepthTexture.Sample(ShadowUV)
  │   float ReceiverDepth = ShadowPos.z（受光面の深度）
  │   float ShadowFactor = ReceiverDepth - Bias < ShadowDepth ? 1.0 : 0.0
  │   （PCF: 近傍 N サンプルの平均 → 0.0〜1.0）
  │
  ├─ [5] カスケードフェード（CSM）
  │   ShadowFadeFraction で隣接カスケードとスムーズブレンド
  │   → CascadeSettings.SplitNearFadeRegion で開始、SplitFarFadeRegion で終了
  │
  └─ [6] Shadow Mask テクスチャへ書き込み
      → RenderLight(..., ShadowMaskTexture)
          DeferredLightPixelShaders.usf:
          LightColor *= ShadowMaskTexture.Sample(ScreenUV)
```

---

## Shadow Depth Bias（アクネ対策）

```
Shadow Acne（自己シャドウノイズ）の原因:
  ReceiverDepth ≈ ShadowDepth のとき浮動小数点誤差で判定が不安定

対策:
  Bias = ConstantDepthBias × 近傍勾配（Slope Scale）
  r.Shadow.DepthBias     : ConstantBias の乗数（デフォルト 1.0）
  r.Shadow.SlopeScaleBias: 傾斜面への追加 Bias（デフォルト 1.0）
  Normal Offset Shadow   : 法線方向に少し押し出してサンプル
```

---

## 関連リファレンス

- [[ref_shadow_projection]] — `TShadowProjectionPS` / PCF サンプルパターン
- [[b_shadow_depth]] — Shadow Depth Map 描画
- [[a_shadow_setup]] — `FProjectedShadowInfo` のカスケード設定
