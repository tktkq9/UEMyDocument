# Lumen Reflections（反射）

- 上位: [[02_lumen_overview]]
- 関連: [[c_lumen_tracing]] | [[d_lumen_radiance_cache]]

---

## 概要

Lumen の反射システム。マテリアルの **Roughness（粗さ）** に応じて  
トレースの精度とコストを動的に切り替える。  
テンポラル蓄積と **ReSTIR** によるサンプル再利用でノイズを低減する。

---

## Roughness による分岐

```
Roughness ≈ 0.0（完全鏡面）
  → 高精度トレース（Mesh SDF / HW RT）
  → ダウンスケール少な め
  → テンポラル蓄積期間短め（動きに追従）

Roughness ≈ 0.3〜0.6（光沢面）
  → ダウンスケール大きめ + テンポラル蓄積多め

Roughness > MaxRoughnessToTrace（例: 0.6）
  → Radiance Cache からサンプリング（→ [[d_lumen_radiance_cache]]）
  → トレースしない（コスト ≈ 0）
```

---

## FLumenReflectionTracingParameters

反射トレーシングシェーダーへのバインド構造体。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenReflectionTracingParameters, )

    // 解像度スケール
    FIntPoint ReflectionDownsampleFactorXY;  // 例: (2,2) → 半解像度でトレース

    // レイの距離設定
    float NearFieldMaxTraceDistance;   // 近距離レイの上限距離
    float FarFieldMaxTraceDistance;    // 遠距離レイの上限距離
    float NearFieldSceneRadius;        // 近距離と遠距離の境界

    // バウンス設定
    uint32 MaxReflectionBounces;       // 最大鏡面バウンス数
    uint32 MaxRefractionBounces;       // 最大屈折バウンス数（ガラス等）

    // フレーム間状態
    uint32 ReflectionsStateFrameIndex;     // フレームインデックス（ジッター用）
    uint32 ReflectionsStateFrameIndexMod8; // 8フレーム周期のインデックス
    uint32 ReflectionsRayDirectionFrameIndex; // レイ方向の周期インデックス

    // Roughness カットオフ（Composite 設定）
    float MaxRoughnessToTrace;             // これ以上は Radiance Cache
    float MaxRoughnessToTraceForFoliage;   // 植生専用カットオフ
    float InvRoughnessFadeLength;          // フェードのシャープさ
    float ReflectionSmoothBias;            // Roughness 補正バイアス

    // レイバッファ（トレース前・後）
    Texture2D~float4~ RayBuffer;          // レイ方向・距離
    Texture2D~float4~ TraceHit;           // ヒット情報
    Texture2D~float4~ TraceRadiance;      // ヒット輝度
    Texture2D~uint~   RayTraceDistance;   // トレース距離（パック済み）

    // ダウンスケール用バッファ
    Texture2D DownsampledDepth;
    Texture2D DownsampledClosureIndex;    // Substrate 対応

    // Pre-integrated GF（BRDF の事前積分）
    Texture2D PreIntegratedGF;
    SamplerState PreIntegratedGFSampler;

END_SHADER_PARAMETER_STRUCT()
```

---

## FReflectionTemporalState（テンポラル状態）

```cpp
class FReflectionTemporalState {
    // 反射輝度の履歴（前フレームのSpecular + 2次モーメント）
    TRefCountPtr<IPooledRenderTarget> SpecularAndSecondMomentHistory;
    // 蓄積フレーム数（多いほどノイズが少ない）
    TRefCountPtr<IPooledRenderTarget> NumFramesAccumulatedHistory;

    // 半透明前面レイヤー専用（Front Layer Transparency）
    TRefCountPtr<IPooledRenderTarget> LayerSceneDepthHistory;
    TRefCountPtr<IPooledRenderTarget> LayerSceneNormalHistory;

    // 再投影用の状態
    uint32 HistoryFrameIndex;
    FIntRect HistoryViewRect;
    FVector4f HistoryScreenPositionScaleBias;
    FVector4f HistoryUVMinMax;
    FVector4f HistoryBufferSizeAndInvSize;
};
```

### テンポラル蓄積の動作

```
1. 現フレームのトレース結果（TraceRadiance）
2. 前フレームの SpecularAndSecondMomentHistory を再投影
3. ブレンド（指数移動平均 or Variance Clipping）
4. 静止しているピクセル → 蓄積フレーム数増加 → ノイズ低減
5. 動いているピクセル → 履歴を棄てる → ゴースト防止
```

---

## ReSTIR（Reservoir-based Spatiotemporal Importance Resampling）

`LumenReSTIRGather.cpp` / `FReSTIRGatherTemporalState`

### ReSTIR とは

少ないレイで高品質なサンプルを得るための **サンプル再利用技術**。  
良いサンプル（明るい・重要な方向）を Reservoir に貯め、次フレームでも再利用する。

```
従来:
  フレームAでサンプルS1を取得 → 使い捨て
  フレームBで新たにS2を取得 → 独立したノイズ

ReSTIR:
  フレームAでS1を取得 → Reservoir に保存（weight付き）
  フレームBで S1 と S2 を比較 → 重みが大きい方を採用
  良いサンプルが長く使われる → ノイズ大幅削減
```

### FReSTIRGatherTemporalState

```cpp
class FReSTIRGatherTemporalState {
    FReSTIRTemporalResamplingState TemporalResamplingState;
    FReSTIRTemporalAccumulationState TemporalAccumulationState;
};

// 時間方向のリサンプリング状態
class FReSTIRTemporalResamplingState {
    TRefCountPtr<IPooledRenderTarget> TemporalReservoirRayDirectionRT;
    TRefCountPtr<IPooledRenderTarget> TemporalReservoirTraceRadianceRT;
    TRefCountPtr<IPooledRenderTarget> TemporalReservoirTraceHitDistanceRT;
    TRefCountPtr<IPooledRenderTarget> TemporalReservoirTraceHitNormalRT;
    TRefCountPtr<IPooledRenderTarget> TemporalReservoirWeightsRT;
    // 再投影用の前フレーム深度・法線
    TRefCountPtr<IPooledRenderTarget> DownsampledDepthHistoryRT;
    TRefCountPtr<IPooledRenderTarget> DownsampledNormalHistoryRT;
};

// 時間方向の蓄積状態
class FReSTIRTemporalAccumulationState {
    TRefCountPtr<IPooledRenderTarget> DiffuseIndirectHistoryRT;
    TRefCountPtr<IPooledRenderTarget> RoughSpecularIndirectHistoryRT;
    TRefCountPtr<IPooledRenderTarget> ResolveVarianceHistoryRT;
    TRefCountPtr<IPooledRenderTarget> NumFramesAccumulatedRT;
};
```

---

## 反射パスの種類（ELumenReflectionPass）

```cpp
// LumenSceneData.h より
enum class ELumenReflectionPass {
    Opaque,                  // 不透明サーフェス
    SingleLayerWater,        // 水面
    FrontLayerTranslucency,  // 半透明の前面レイヤー
    MAX
};
```

---

## LumenReflections 名前空間

```cpp
// 合成用パラメータ
namespace LumenReflections {
    struct FCompositeParameters {
        float MaxRoughnessToTrace;
        float MaxRoughnessToTraceForFoliage;
        float InvRoughnessFadeLength;
        float ReflectionSmoothBias;
    };

    // 非同期 Compute で反射を実行できるか判定
    bool UseAsyncCompute(const FViewFamilyInfo&,
                         EDiffuseIndirectMethod,
                         EReflectionsMethod);
}
```

---

## 主要 CVar

```
# 反射全体
r.Lumen.Reflections.Allow = 1
r.Lumen.Reflections.MaxRoughnessToTrace = 0.4
    ← これ以上の Roughness は Radiance Cache で代替

# トレース精度
r.Lumen.Reflections.DownsampleFactor = 1
    ← 1:フル解像度  2:半解像度  ...（コスト vs 品質）

# バウンス
r.Lumen.Reflections.MaxBounces = 2
r.Lumen.Reflections.MaxRefractionBounces = 3

# ReSTIR
r.Lumen.Reflections.ReSTIR.Enable = 1

# テンポラル
r.Lumen.Reflections.Temporal.MaxFramesAccumulated
```

---

## コード実行フロー

### エントリポイント

反射は Lighting フェーズの `RenderDiffuseIndirectAndAmbientOcclusion()` から呼ばれる。

```
RenderDiffuseIndirectAndAmbientOcclusion()   (IndirectLightRendering.cpp:977)
  │
  └─ [ReflectionsMethod == Lumen 時]
      RenderLumenReflections(GraphBuilder, View, SceneTextures, FrameTemporaries,
                             MeshSDFGridParameters, RadianceCacheParameters,
                             ELumenReflectionPass::Opaque, ...)
        │                    (LumenReflections.cpp:1154)
        │
        ├─ SetupCompositeParameters()          ← MaxRoughnessToTrace 等の合成パラメータ設定
        │
        ├─ ReflectionTileClassification(...)   ← タイル分類（Roughness・有効ピクセル判定）
        │
        ├─ FReflectionGenerateRaysCS           ← レイ方向バッファを生成（DownsampledDepth も）
        │
        ├─ TraceReflections(...)               ← SW RT パス（LumenReflectionTracing.cpp:1109）
        │   または
        │   RenderLumenHardwareRayTracingReflections(...)  ← HW RT パス
        │
        ├─ FLumenReflectionResolveCS           ← ヒット輝度を解像度に展開（空間再構成）
        │
        ├─ [bTemporal == true 時]
        │   FLumenReflectionDenoiserTemporalCS ← テンポラル蓄積（前フレーム再投影）
        │   FLumenReflectionDenoiserSpatialCS  ← 空間バイラテラルフィルタ
        │
        └─ SpecularIndirect を返す

[追加パス - 別途 RenderLumenReflections() を呼ぶ]
  ├─ RenderLumenFrontLayerTranslucencyReflections(...)   ← 半透明前面レイヤーの反射
  │   └─ ELumenReflectionPass::FrontLayerTranslucency
  └─ SingleLayerWater → ELumenReflectionPass::SingleLayerWater
```

### フロー詳細

1. **ReflectionTileClassification** — Roughness と有効ピクセルでタイルを3分類（`LumenReflections.cpp:1285`）
   ```cpp
   ReflectionTileParameters = ReflectionTileClassification(
       GraphBuilder, View, SceneTextures, FrameTemporaries,
       ReflectionTracingParameters, ReflectionsConfig.FrontLayerReflectionGBuffer,
       ComputePassFlags);
   ```
   - Clear / Resolve / Tracing の3種の IndirectArgs を生成
   - 参照: [[ref_lumen_reflections]]

2. **FReflectionGenerateRaysCS** — GBuffer から反射レイを生成、DownsampledDepth と RayBuffer に書き込む（`LumenReflections.cpp:1319`）
   ```cpp
   // DownsampledDepth（半解像度）と RayBuffer（方向・距離）を生成
   FReflectionGenerateRaysCS::FParameters* PassParameters = ...;
   PermutationVector.Set<FReflectionGenerateRaysCS::FRadianceCache>(bUseRadianceCache);
   PermutationVector.Set<FReflectionGenerateRaysCS::FFrontLayerTranslucency>(bFrontLayer);
   ```
   - `MaxRoughnessToTrace` 以上の Roughness は Radiance Cache に委譲
   - 参照: [[ref_lumen_reflection_tracing]]

3. **TraceReflections / RenderLumenHardwareRayTracingReflections** — レイトレースパス（`LumenReflections.cpp:1400`）
   ```cpp
   TraceReflections(GraphBuilder, Scene, View, FrameTemporaries,
       bTraceMeshObjects, SceneTextures, ReflectionTracingParameters,
       ReflectionTileParameters, MeshSDFGridParameters, bUseRadianceCache,
       DiffuseIndirectMethod, RadianceCacheParameters, ...);
   ```
   - SW RT: Mesh SDF → Global SDF → FarField → RadianceCache の多段フォールバック
   - HW RT: `UseHardwareRayTracedReflections()` が true の場合に Inline RT / RGS を使用
   - 参照: [[ref_lumen_reflection_tracing]] | [[ref_lumen_reflection_hwrt]]

4. **FLumenReflectionResolveCS → テンポラル → 空間フィルタ** — 解像度展開・ノイズ除去（`LumenReflections.cpp:1513, 1652, 1710`）
   ```cpp
   // Resolve: ダウンスケールしたトレース結果をフル解像度に展開
   FLumenReflectionResolveCS: bUseSpatialReconstruction で近傍サンプルを利用

   // Temporal: 前フレームの SpecularAndSecondMomentHistory と指数移動平均
   FLumenReflectionDenoiserTemporalCS: PermutationVector.Set<FValidHistory>(...);

   // Spatial: バイラテラルフィルタで残存ノイズを平滑化
   FLumenReflectionDenoiserSpatialCS: bSpatial = GLumenReflectionBilateralFilter != 0;
   ```
   - 参照: [[ref_lumen_reflections]]

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|------------|--------|------|
| `RenderLumenReflections()` | `LumenReflections.cpp` | 反射のトップレベルエントリ（全パス共通） |
| `ReflectionTileClassification()` | `LumenReflections.cpp` | Roughness によるタイル分類 |
| `FReflectionGenerateRaysCS` | `LumenReflections.cpp` | 反射レイ方向バッファの生成 |
| `TraceReflections()` | `LumenReflectionTracing.cpp` | SW RT トレースパス |
| `RenderLumenHardwareRayTracingReflections()` | `LumenReflectionHardwareRayTracing.cpp` | HW RT トレースパス |
| `FLumenReflectionResolveCS` | `LumenReflections.cpp` | トレース結果の解像度展開・空間再構成 |
| `FLumenReflectionDenoiserTemporalCS` | `LumenReflections.cpp` | テンポラル蓄積デノイザー |
| `FLumenReflectionDenoiserSpatialCS` | `LumenReflections.cpp` | 空間バイラテラルフィルタ |
| `FReflectionTemporalState` | `LumenViewState.h` | 反射の前フレーム履歴（SpecularAndSecondMomentHistory 等） |

---

## 関連ソースファイル

| ファイル | 役割 |
|---------|------|
| `LumenReflections.h/cpp` | 反射メインパス・LumenReflections名前空間 |
| `LumenReflectionTracing.cpp` | レイトレースパス |
| `LumenReflectionHardwareRayTracing.cpp` | HW RT バリアント |
| `LumenReSTIRGather.h/cpp` | ReSTIR 実装 |
| `LumenFrontLayerTranslucency.h/cpp` | 半透明前面レイヤーの反射 |
| `LumenViewState.h` | FReflectionTemporalState / FReSTIRGatherTemporalState |

---

## 関連リファレンス

| リファレンス | 対象ソース |
|------------|----------|
| [[ref_lumen_reflections]] | `LumenReflections.h/cpp` |
| [[ref_lumen_reflection_tracing]] | `LumenReflectionTracing.cpp` |
| [[ref_lumen_reflection_hwrt]] | `LumenReflectionHardwareRayTracing.cpp` |
| [[ref_lumen_restir]] | `LumenReSTIRGather.h/cpp` |
| [[ref_lumen_front_layer]] | `LumenFrontLayerTranslucency.h/cpp` |
| [[ref_ray_traced_translucency]] | `RayTracedTranslucency.h` |
