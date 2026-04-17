# ref: TShadowProjectionPS / FShadowProjectionPixelShaderInterface / PCSS

- 対象ファイル: `ShadowRendering.h` / `ShadowRendering.cpp`
- 概要: [[17_shadow_overview]]

---

## FShadowProjectionPixelShaderInterface（基底クラス）

```cpp
// ShadowRendering.h
class FShadowProjectionPixelShaderInterface : public FGlobalShader
{
    // 全 Shadow Projection PS の基底クラス
    // SetParameters() を純粋仮想として定義

    void SetParameters(
        FRHIBatchedShaderParameters& BatchedParameters,
        int32 ViewIndex,
        const FSceneView& View,
        const FProjectedShadowInfo* ShadowInfo,
        bool bUseLightFunctionAtlas);

    // ─── バインドされるシェーダーパラメータ ─────────────────────
    FShaderParameter ScreenToShadowMatrix;  // スクリーン座標 → Shadow UV
    FShaderParameter SoftTransitionScale;   // ソフト遷移スケール
    FShaderParameter ShadowBufferSize;      // Shadow Atlas サイズ（テクセル単位）
    FShaderResourceParameter ShadowDepthTexture;        // Shadow Depth Atlas
    FShaderResourceParameter ShadowDepthTextureSampler;
    FShaderParameter ProjectionDepthBias;   // Shadow Acne 対策バイアス
    FShaderParameter FadePlaneOffset;       // カスケードフェード平面オフセット
    FShaderParameter InvFadePlaneLength;    // フェード長さの逆数
    FShaderParameter ShadowTileOffsetAndSizeParam; // Atlas 内タイル位置とサイズ
    FShaderParameter LightPositionOrDirection;      // ライスト位置 / 方向
    FShaderParameter PerObjectShadowFadeStart;
    FShaderParameter InvPerObjectShadowFadeLength;
    FShaderParameter ShadowNearAndFarDepth; // SubPixel シャドウ用カスケード深度範囲
    FShaderParameter bCascadeUseFadePlane;
};
```

---

## TShadowProjectionPS テンプレート

```cpp
// ShadowRendering.h:1236
template<
    uint32 Quality,                  // PCF サンプル数品質 (1/2/4/8/16/32)
    bool bUseFadePlane = false,      // カスケード間フェード（隣接カスケード補間）
    bool bModulatedShadows = false,  // モバイル向け乗算合成シャドウ
    bool bUseTransmission = false,   // バックライト透過シャドウ
    bool SubPixelShadow = false      // サブピクセルシャドウ（Hair Strands 向け）
>
class TShadowProjectionPS : public FShadowProjectionPixelShaderInterface
{
    // Quality 別のサンプル数:
    // Quality=1  →  1 サンプル（最軽量）
    // Quality=2  →  4 サンプル
    // Quality=3  →  16 サンプル
    // Quality=4  →  16 サンプル（PCSS 用）
    // Quality=5  →  32 サンプル（最高品質）

    void SetParameters(
        FRHIBatchedShaderParameters& BatchedParameters,
        int32 ViewIndex,
        const FSceneView& View,
        const FProjectedShadowInfo* ShadowInfo,
        bool bUseLightFunctionAtlas);

private:
    FShaderParameter ShadowFadeFraction;  // カスケードフェード係数
    FShaderParameter ShadowSharpen;       // シャドウ鮮明度
    FShaderParameter LightPosition;       // LightPositionAndInvRadius
};
```

---

## PCSS 専用派生クラス

```cpp
// Directional Light 向け PCSS
template<uint32 Quality, bool bUseFadePlane>
class TDirectionalPercentageCloserShadowProjectionPS
    : public TShadowProjectionPS<Quality, bUseFadePlane>
{
    // ModifyCompilationEnvironment: USE_PCSS=1 を定義
    // PCSSParameters を追加バインド
    //   → TanLightSourceAngle（= LightSourceAngle の半値の Tan）
    //   → MaxSoftKernelSize (r.Shadow.MaxSoftKernelSize)
};

// Spot Light 向け PCSS
template<uint32 Quality, bool bUseFadePlane>
class TSpotPercentageCloserShadowProjectionPS
    : public TShadowProjectionPS<Quality, bUseFadePlane>
{
    // ModifyCompilationEnvironment: USE_PCSS=1, SPOT_LIGHT_PCSS=1 を定義
};

// モバイル向け乗算合成
template<uint32 Quality>
class TModulatedShadowProjection
    : public TShadowProjectionPS<Quality, false, true>
{
    // ModulatedShadowColor を追加バインド
};

// 半透明シャドウ対応
template<uint32 Quality>
class TShadowProjectionFromTranslucencyPS
    : public TShadowProjectionPS<Quality>
{
    // APPLY_TRANSLUCENCY_SHADOWS=1
    // FTranslucentSelfShadowUniformParameters をバインド
};

// Point Light（1パス・キューブマップ）
template <uint32 Quality, bool bUseTransmission, bool bUseSubPixel>
class TOnePassPointShadowProjectionPS : public FGlobalShader
{
    // OnePassPoint キューブマップ専用
    // ShadowDepthCubeTexture を使用
};
```

---

## PCF アルゴリズム（ShadowProjectionPixelShader.usf）

```
PCF（Percentage Closer Filtering）:

シェーダー内でのサンプリング処理:
  1. ScreenUV → ShadowUV 変換（ScreenToShadowMatrix）
  2. ShadowDepth = ShadowDepthTexture.SampleLevel(ShadowUV, 0)
  3. ReceiverDepth = ShadowUV.z（ライスト空間の深度）
  4. ShadowFactor = ReceiverDepth - ProjectionDepthBias < ShadowDepth ? 1.0 : 0.0

PCF 近傍サンプリング（Quality に応じたパターン）:
  Quality=1: 中心1サンプルのみ（ハードシャドウ）
  Quality=2: 2×2 サンプル（4点グリッド）
  Quality=3: Poisson ディスク 16 サンプル
  Quality=4: Poisson ディスク 16 サンプル（PCSS 用半径スケール対応）
  Quality=5: Poisson ディスク 32 サンプル

最終 ShadowFactor = sum(SampleResults) / Quality
  → 0.0: 完全シャドウ
  → 1.0: 完全ライット
```

---

## PCSS アルゴリズム詳細

```
PCSS（Percentage Closer Soft Shadows）:
USE_PCSS=1 時（TDirectionalPercentageCloserShadowProjectionPS / TSpotPercentageCloserShadowProjectionPS）

[Step 1] Blocker Search
  r.Shadow.PCSS.BlockerSearchNumSamples 個でサンプル
  ShadowMap をサンプルして遮蔽物深度を収集
  d_blocker = 遮蔽物の平均深度

[Step 2] Penumbra Radius 計算
  PenumbraRadius = (d_receiver - d_blocker) × LightSourceAngle / d_blocker
  → 遮蔽物が近い: 小さい半径 → ハードシャドウ
  → 遮蔽物が遠い: 大きい半径 → ソフトシャドウ

  MaxSoftKernelSize で上限クランプ (r.Shadow.MaxSoftKernelSize)

[Step 3] PCF with variable radius
  PenumbraRadius に応じたサンプル半径で PCF 実行
  → ShadowFactor (0.0〜1.0)
```

---

## Shadow Projection ディスパッチ選択（ShadowRendering.cpp）

```cpp
// r.Shadow.FilterMethod に応じた Projection PS の選択
switch (r.Shadow.FilterMethod)
{
case 0:  // PCF
    TShadowProjectionPS<Quality>                       // 通常
    TDirectionalPercentageCloserShadowProjectionPS     // Directional（bUseFadePlane）
    TSpotPercentageCloserShadowProjectionPS            // Spot（bUseFadePlane）
    break;

case 1:  // PCSS
    USE_PCSS=1 で同上の PCSS 派生クラスを選択
    break;
}

// Quality 選択（r.Shadow.Quality）:
// 0→ Quality=1, 1→ Quality=2, 2→ Quality=3, 3→ Quality=4（デフォルト）, 4→ Quality=5
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Shadow.FilterMethod` | 0 | 0=PCF, 1=PCSS |
| `r.Shadow.Quality` | 3 | PCF サンプル品質（0-4）|
| `r.Shadow.DepthBias` | 1.0 | 深度バイアス乗数（Shadow Acne 対策）|
| `r.Shadow.SlopeScaleBias` | 1.0 | 傾斜面追加バイアス |
| `r.Shadow.PCSS.BlockerSearchNumSamples` | 16 | PCSS ブロッカーサーチサンプル数 |
| `r.Shadow.MaxSoftKernelSize` | 40 | PCSS 最大カーネルサイズ（テクセル）|

---

## 関連リファレンス

- [[ref_projected_shadow_info]] — `FProjectedShadowInfo` 全メンバ
- [[ref_shadow_rendering]] — `FShadowDepthVS` / `FShadowDepthMeshProcessor`
- [[c_shadow_projection]] — Shadow Projection 詳細フロー
