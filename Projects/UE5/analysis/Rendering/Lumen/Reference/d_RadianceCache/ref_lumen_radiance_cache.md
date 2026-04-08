# リファレンス：LumenRadianceCache.h / LumenRadianceCache.cpp

- グループ: d - Radiance Cache
- 上位: [[d_lumen_radiance_cache]]
- 関連: [[ref_lumen_radiance_cache_internal]] | [[ref_lumen_tracing_utils]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenRadianceCache.h/cpp`

---

## 概要

Lumen の **Radiance Cache** システムの公開 API。  
World Space にクリップマップ形式で配置した放射照度プローブを管理し、  
Screen Probe Gather・Reflections・透明マテリアル照明などの間接照明に提供する。  
`UpdateRadianceCaches()` が 1 フレームの更新エントリポイントで、  
複数のキャッシュ（GI 用・反射用・透明用など）を同時にオーバーラップ更新して GPU 効率を上げる。

---

## 定数

```cpp
namespace LumenRadianceCache {
    static constexpr int32 MaxClipmaps = 6;              // 最大クリップマップ数
    static constexpr int32 MinRadianceProbeResolution = 8; // プローブ最小解像度
}
```

---

## FRadianceCacheConfiguration

Radiance Cache の動作オプション設定。

```cpp
struct FRadianceCacheConfiguration {
    bool bFarField    = true;  // Far Field トレースを使うか
    bool bSkyVisibility = false; // Sky Visibility を計算するか
};
```

---

## FUpdateInputs

`UpdateRadianceCaches()` に渡す**読み取り専用入力**をまとめたクラス。

```cpp
class FUpdateInputs {
public:
    FRadianceCacheInputs    RadianceCacheInputs;   // プローブ解像度・クリップマップ設定
    FRadianceCacheConfiguration Configuration;     // Far Field / SkyVisibility フラグ
    const FViewInfo& View;

    // Screen Probe パラメータ（ScreenProbeGather との連携）
    const FScreenProbeParameters* ScreenProbeParameters;

    // BRDF 重要度サンプリング SH（重要度サンプリング用）
    FRDGBufferSRVRef BRDFProbabilityDensityFunctionSH;

    // プローブを「使用済み」としてマークするデリゲート（Graphics/Compute 別）
    FMarkUsedRadianceCacheProbes GraphicsMarkUsedRadianceCacheProbes;
    FMarkUsedRadianceCacheProbes ComputeMarkUsedRadianceCacheProbes;
};
```

---

## FUpdateOutputs

`UpdateRadianceCaches()` の**出力**をまとめたクラス。

```cpp
class FUpdateOutputs {
public:
    FRadianceCacheState& RadianceCacheState;                        // 永続状態（フレーム間キャッシュ）
    FRadianceCacheInterpolationParameters& RadianceCacheParameters; // 補間用シェーダーパラメータ
};
```

---

## FRadianceCacheMarkParameters

使用するプローブを 3D インディレクションテクスチャにマークするシェーダーパラメータ。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FRadianceCacheMarkParameters, )
    SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture3D<uint>, RWRadianceProbeIndirectionTexture)
    SHADER_PARAMETER_ARRAY(FVector4f, ClipmapCornerTWSAndCellSizeForMark, [MaxClipmaps])
    SHADER_PARAMETER(uint32, RadianceProbeClipmapResolutionForMark)
    SHADER_PARAMETER(uint32, NumRadianceProbeClipmapsForMark)
    SHADER_PARAMETER(float, InvClipmapFadeSizeForMark)
END_SHADER_PARAMETER_STRUCT()
```

---

## マルチキャストデリゲート

```cpp
// プローブを「使用済み」としてマークする処理を外部から差し込むデリゲート
// → Screen Probe / Translucency / Visualization など複数のシステムが登録できる
DECLARE_MULTICAST_DELEGATE_ThreeParams(
    FMarkUsedRadianceCacheProbes,
    FRDGBuilder&,
    const FViewInfo&,
    const LumenRadianceCache::FRadianceCacheMarkParameters&);
```

---

## 主要関数（LumenRadianceCache 名前空間）

```cpp
namespace LumenRadianceCache {

    // 複数の Radiance Cache を 1 フレームでまとめて更新
    // GPU ディスパッチをオーバーラップさせて効率を最大化
    void UpdateRadianceCaches(
        FRDGBuilder& GraphBuilder,
        const FLumenSceneFrameTemporaries& FrameTemporaries,
        const TInlineArray<FUpdateInputs>& InputArray,   // 複数キャッシュの入力
        TInlineArray<FUpdateOutputs>& OutputArray,       // 複数キャッシュの出力
        const FScene* Scene,
        const FViewFamilyInfo& ViewFamily,
        bool bPropagateGlobalLightingChange,
        ERDGPassFlags ComputePassFlags);

    // Lumen シーンライティングの Compute パスフラグを取得
    ERDGPassFlags GetLumenSceneLightingComputePassFlags(const FEngineShowFlags& EngineShowFlags);

    // HW RT の Hit Lighting を使うかどうか
    bool UseHitLighting(const FViewInfo& View, EDiffuseIndirectMethod DiffuseIndirectMethod);
}

// デバッグ用: Radiance Cache プローブの可視化マーク
extern void MarkUsedProbesForVisualize(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const LumenRadianceCache::FRadianceCacheMarkParameters& RadianceCacheMarkParameters,
    ERDGPassFlags ComputePassFlags);
```

---

## 主要 CVar（LumenRadianceCache.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.RadianceCache.Update` | 1 | 毎フレーム更新するか（0=停止、デバッグ用）|
| `r.Lumen.RadianceCache.ForceFullUpdate` | 0 | 全プローブを一度に更新する（デバッグ用）|
| `r.Lumen.RadianceCache.NumFramesToKeepCachedProbes` | 8 | 未使用プローブを保持するフレーム数（多いほど再利用が増えるが古くなる）|
| `r.Lumen.RadianceCache.OverrideCacheOcclusionLighting` | 0 | オクルージョン照明を上書きするか（デバッグ）|
| `r.Lumen.RadianceCache.ShowBlackRadianceCacheLighting` | 0 | 黒い照明を表示（デバッグ）|
| `r.Lumen.RadianceCache.SpatialFilterProbes` | 1 | 近傍プローブ間の空間フィルタリング |
| `r.Lumen.RadianceCache.SortTraceTiles` | 0 | トレースタイルを方向でソートして局所性向上 |
| `r.Lumen.RadianceCache.SpatialFilterMaxRadianceHitAngle` | 0.2 | フィルタリング時の最大ヒット角度（度）|

---

## Radiance Cache の更新フロー

```
UpdateRadianceCaches()
  │
  ├─ [Mark Pass] MarkUsedRadianceCacheProbes デリゲートを呼ぶ
  │   → Screen Probe Gather が「このプローブを使う」とマーク
  │   → Translucency が使用プローブをマーク
  │
  ├─ [Allocate] 使用プローブのインデックスをアトラスに割り当て
  │   → FRadianceCacheState でフレーム間の配置を管理
  │   → NumFramesToKeepCachedProbes フレーム以内なら再利用
  │
  ├─ [Trace Budget] NumProbesToTraceBudget 個のプローブを選択
  │   → 古い順・重要度順に選択
  │
  ├─ [Trace Pass] 選択されたプローブをトレース
  │   → SW RT: Surface Cache から Radiance をサンプリング
  │   → HW RT: RenderLumenHardwareRayTracingRadianceCache() を呼ぶ
  │
  └─ [Filter & SH] プローブ Radiance を SH に射影・空間フィルタ
        → FRadianceCacheInterpolationParameters に格納
        → Screen Probe Gather / Translucency が補間して使用
```
