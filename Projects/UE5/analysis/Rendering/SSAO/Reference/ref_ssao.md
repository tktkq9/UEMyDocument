# ref: FGTAOContext / FSSAOHelper / EGTAOType / EGTAOPass

- 対象ファイル: `PostProcessAmbientOcclusion.h/.cpp`
- 概要: [[22_ssao_overview]]

---

## ESSAOType（PostProcessAmbientOcclusion.h:19）

```cpp
enum class ESSAOType
{
    EPS,       // ピクセルシェーダー版（古い SM5 デバイス）
    ECS,       // 非 Async Compute シェーダー版
    EAsyncCS,  // Async Compute シェーダー版（推奨）
};
```

---

## EGTAOType（PostProcessAmbientOcclusion.h:29）

```cpp
enum class EGTAOType
{
    EOff,                    // GTAO 無効（SSAO レガシーを使用）

    // HorizonSearch + Integrate を Async Compute で実行
    // Temporal / Upsample は GFX パイプ（Velocity Buffer が必要なため）
    EAsyncCombinedSpatial,

    // HorizonSearch のみ Async Compute
    // Integrate 以降は GFX パイプ（GBuffer チャンネルが必要なため）
    EAsyncHorizonSearch,

    // 全パスを GFX パイプで実行（Async 非対応プラットフォーム）
    ENonAsync,
};
```

---

## EGTAOPass（PostProcessAmbientOcclusion.h:45）

```cpp
enum EGTAOPass
{
    EGTAOPass_None                  = 0x0,
    EGTAOPass_HorizonSearch         = 0x1,  // 水平線探索
    EGTAOPass_HorizonSearchIntegrate = 0x2, // 探索＋積分（Combined パス）
    EGTAOPass_Integrate             = 0x4,  // 立体角積分（Separate パス）
    EGTAOPass_SpatialFilter         = 0x8,  // 空間バイラテラルフィルタ
    EGTAOPass_TemporalFilter        = 0x10, // 時間的フィルタ
    EGTAOPass_Upsample              = 0x20, // ハーフ解像度アップサンプル
};
```

---

## FGTAOContext（PostProcessAmbientOcclusion.h:63）

```cpp
class FGTAOContext
{
public:
    EGTAOType GTAOType;         // 実行モード
    uint32 FinalPass;           // 最終パスの EGTAOPass 値
    uint32 DownsampleFactor;    // 1=フル解像度, 2=ハーフ解像度

    bool bUseNormals;           // GBufferA から法線を読むか（精度向上）
    bool bHalfRes;              // ハーフ解像度実行（r.GTAO.HalfRes=1）
    bool bHasSpatialFilter;
    bool bHasTemporalFilter;

    FGTAOContext(EGTAOType Type);
    FGTAOContext();

    bool IsFinalPass(EGTAOPass Pass);
    // → Pass == FinalPass なら true（書き込み先を切り替えるために使用）
};
```

---

## FSSAOHelper（PostProcessAmbientOcclusion.h:82）

```cpp
class FSSAOHelper
{
    // AO 品質パラメータ（ビュー設定 + CVar の最小値）
    static float GetAmbientOcclusionQualityRT(const FSceneView& View);
    // → PostProcessSettings.AmbientOcclusionQuality × r.AmbientOcclusion.Quality

    // シェーダー品質レベル（0〜4）
    static int32 GetAmbientOcclusionShaderLevel(const FSceneView& View);
    // → Level 0: 1 サンプル（最軽量）
    // → Level 4: 最大サンプル数

    // Compute Shader を使うべきか（SM6 以上なら CS 推奨）
    static bool IsAmbientOcclusionCompute(const ERHIFeatureLevel::Type FeatureLevel);

    static int32 GetNumAmbientOcclusionLevels();
    static float GetAmbientOcclusionStepMipLevelFactor();
    static EAsyncComputeBudget GetAmbientOcclusionAsyncComputeBudget();

    // BasePass AO が必要か（Substrate 等の特殊ケース）
    static bool IsBasePassAmbientOcclusionRequired(const FViewInfo& View);
};
```

---

## AO テクスチャ管理（PostProcessAmbientOcclusion.h）

```cpp
// AO テクスチャのフォーマット定義
FRDGTextureDesc GetScreenSpaceAOTextureDesc(
    ERHIFeatureLevel::Type FeatureLevel,
    FIntPoint Extent);
// → SM6: R8G8（AO + BentNormal.x）
//    SM5: R8（AO のみ）

// AO テクスチャを生成
FRDGTextureRef CreateScreenSpaceAOTexture(
    FRDGBuilder& GraphBuilder,
    ERHIFeatureLevel::Type FeatureLevel,
    FIntPoint Extent);

// AO が無効な場合のフォールバック（白 = 遮蔽なし）
FRDGTextureRef GetScreenSpaceAOFallback(const FRDGSystemTextures& SystemTextures);
```

---

## GTAO シェーダー（GTAO.usf）

```
【HorizonSearch CS（GTAOHorizonSearch.usf）】
  入力: SceneDepth, GBufferA（法線）
  出力: HorizonAngles Texture（R16G16F）
  各ピクセルで:
    for DirectionIndex in [0, NumDirections):
      Step 方向に SSAO_SAMPLES_NUM 回サンプル
      → 最大仰角 h1（+方向）/ h2（-方向）を探索
      → HorizonAngle = (h1, h2)

【Integrate CS（GTAOIntegrate.usf）】
  入力: HorizonAngles, GBufferA（法線）
  出力: IntegratedAO（R8G8: AO + BentNormal）
  各ピクセルで:
    for DirectionIndex:
      N = GBufferNormal
      AO = integral(cos_theta, h1〜h2) / π（数値積分）
      BentNormal += 非遮蔽方向の寄与

【SpatialFilter CS（GTAOSpatialFilter.usf）】
  バイラテラルフィルタ:
    Weight = gaussian(|depth_diff| / DepthSigma)
           × gaussian(dot(N_center, N_sample) / NormalSigma)

【TemporalFilter CS（GTAOTemporalFilter.usf）】
  VelocityTexture で前フレーム UV を計算（再投影）
  Neighborhood Clamp（前フレーム値を現在近傍の min-max にクランプ）
  AO_out = lerp(AO_current, AO_history_clamped, 1 / (1 + AccumCount))
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.AmbientOcclusion.Method` | 1 | 0=SSAO, 1=GTAO |
| `r.GTAO.Quality` | 3 | 品質（0-4, サンプル数に対応）|
| `r.GTAO.HalfRes` | 1 | 1/2 解像度実行 |
| `r.GTAO.UseNormals` | 1 | GBuffer 法線使用 |
| `r.GTAO.TemporalFilter` | 1 | Temporal Filter 有効 |
| `r.GTAO.Async` | 1 | Async Compute 使用 |

---

## 関連リファレンス

- [[ref_composition_lighting]] — `FCompositionLighting` / PostProcess 統合
- [[a_ssao]] — GTAO / SSAO 詳細フロー
