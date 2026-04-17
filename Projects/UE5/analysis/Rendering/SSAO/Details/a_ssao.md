# a: SSAO / GTAO パス（AddAmbientOcclusionPasses）

- 対象ファイル: `PostProcessAmbientOcclusion.h/.cpp`
- 概要: [[22_ssao_overview]]

---

## 概要

UE5 の Screen Space AO は主に **GTAO**（Ground Truth Ambient Occlusion）を使用する。  
Horizon Search で水平線角度を探索し、立体角積分で AO 値を計算する物理ベースの手法。  
Async Compute で Spatial/Temporal を分離実行してパイプラインを効率化する。

---

## FGTAOContext（PostProcessAmbientOcclusion.h）

```cpp
class FGTAOContext
{
public:
    EGTAOType GTAOType;           // Async/NonAsync の選択
    uint32 FinalPass;             // 最終パスの EGTAOPass ビットマスク
    uint32 DownsampleFactor;      // 1 = フル解像度, 2 = ハーフ解像度

    bool bUseNormals;             // GBuffer 法線を使用するか
    bool bHalfRes;                // ハーフ解像度で実行するか
    bool bHasSpatialFilter;       // Spatial Filter を実行するか
    bool bHasTemporalFilter;      // Temporal Filter を実行するか（Velocity 必要）

    FGTAOContext(EGTAOType Type); // EGTAOType からコンテキストを初期化
    bool IsFinalPass(EGTAOPass);  // このパスが最終パスかどうか
};
```

---

## FSSAOHelper（PostProcessAmbientOcclusion.h）

```cpp
class FSSAOHelper
{
    static float GetAmbientOcclusionQualityRT(const FSceneView& View);
    // → PostProcessSettings.AmbientOcclusionQuality + r.AmbientOcclusion.Quality の最小値

    static int32 GetAmbientOcclusionShaderLevel(const FSceneView& View);
    // → 0〜4 のシェーダー品質レベル（サンプル数に対応）

    static bool IsAmbientOcclusionCompute(const ERHIFeatureLevel::Type FeatureLevel);
    // → SM6 以上なら Compute Shader を使用

    static bool IsBasePassAmbientOcclusionRequired(const FViewInfo& View);
    // → BasePass での AO 出力（Substrate 等）が必要な場合

    static EAsyncComputeBudget GetAmbientOcclusionAsyncComputeBudget();
    // → EAsyncComputeBudget::EAll/ELeast など
};
```

---

## GTAO パスフロー（AddAmbientOcclusionPasses）

```
AddAmbientOcclusionPasses(GraphBuilder, View, GTAOContext, ...)
  │
  ├─ [A] HorizonSearch（EGTAOPass_HorizonSearch）
  │   入力: SceneDepth（または HZB Mip0）, GBufferA（法線 ← bUseNormals=true 時）
  │   出力: HorizonSearchTexture（R16G16F）
  │
  │   CS: GTAOHorizonSearch.usf
  │   for each 方向 in [0, NumDirections):
  │     ステップ方向にサンプルしながら最大仰角を探索
  │     h1 = MaxHorizonAngle(+方向)
  │     h2 = MaxHorizonAngle(-方向)
  │     → HorizonAngle (h1, h2) をテクスチャに格納
  │
  ├─ [B] Integrate（EGTAOPass_Integrate）
  │   入力: HorizonSearchTexture
  │   出力: IntegrateTexture（R8G8: AO + BentNormal.xy）
  │
  │   CS: GTAOIntegrate.usf
  │   for each ピクセル:
  │     N = GBufferNormal（法線方向）
  │     AO = 積分(cos(theta), h1〜h2) / π
  │       → 水平線角度の範囲内は遮蔽 → AO 低下
  │     BentNormal = 非遮蔽方向の加重平均
  │
  ├─ [C] SpatialFilter（EGTAOPass_SpatialFilter）
  │   入力: IntegrateTexture
  │   出力: FilteredTexture
  │
  │   CS: GTAOSpatialFilter.usf
  │   バイラテラルフィルタ:
  │     Weight = exp(-depth_diff * DepthScale) × exp(-normal_diff * NormalScale)
  │     → 深度不連続・法線不連続でフィルタを抑制
  │
  ├─ [D] TemporalFilter（EGTAOPass_TemporalFilter）
  │   入力: FilteredTexture, VelocityTexture, HistoryTexture
  │   出力: AOTexture（最終出力）
  │
  │   CS: GTAOTemporalFilter.usf
  │   前フレーム AO を Velocity で再投影
  │   Neighborhood Clamp で過去値をクランプ
  │   BlendFactor = 1 / (1 + NumFrames) → 時間的平均
  │
  └─ [E] Upsample（EGTAOPass_Upsample, bHalfRes=true 時）
      ハーフ解像度 → フル解像度
      バイラテラルアップサンプル（SceneDepth ベース）
      出力: AOTexture（フル解像度）
```

---

## SSAO（従来方式, r.AmbientOcclusion.Method=0）

```
SSAO Setup CS（PostProcessAmbientOcclusionSetup.usf）:
  半球状にサンプル方向を定義（Poisson ディスク状）
  各サンプルの深度を SceneDepth から取得
  → 法線半球内の遮蔽率を計算

SSAO Blur PS（空間フィルタ）:
  縦横の 2 パスブラー（Gaussian 近似）

出力: R8 テクスチャ（AO = 0.0〜1.0）
```

---

## AO テクスチャフォーマット

```cpp
// AO テクスチャの生成
FRDGTextureRef CreateScreenSpaceAOTexture(GraphBuilder, FeatureLevel, Extent)
// → SM6: R8G8（AO + BentNormal.x encoded）
//    SM5: R8（AO のみ）

// フォールバック
FRDGTextureRef GetScreenSpaceAOFallback(SystemTextures)
// → AO=1.0（遮蔽なし）の白テクスチャ（AO が無効な場合）
```

---

## 関連リファレンス

- [[ref_ssao]] — `FGTAOContext` / `FSSAOHelper` / シェーダークラス詳細
- [[b_composition_lighting]] — CompositionLighting での AO 統合
