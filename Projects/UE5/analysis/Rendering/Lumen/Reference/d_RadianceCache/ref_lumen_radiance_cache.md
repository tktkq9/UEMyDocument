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

### 使用箇所

- [[ref_lumen_radiance_cache_internal]] — `FRadianceCacheInputs::NumRadianceProbeClipmaps` の上限チェックに使用
- [[ref_lumen_radiance_cache_internal]] — `FRadianceCacheInterpolationParameters::RadianceProbeSettings[MaxClipmaps]` の配列サイズに使用

---

## FRadianceCacheConfiguration

Radiance Cache の動作オプション設定。

```cpp
struct FRadianceCacheConfiguration {
    bool bFarField    = true;  // Far Field トレースを使うか
    bool bSkyVisibility = false; // Sky Visibility を計算するか
};
```

### メンバ変数

| 変数名 | 型 | デフォルト | 説明 |
|--------|-----|-----------|------|
| `bFarField` | `bool` | `true` | Far Field TLAS を使った遠距離トレースを行うか。GI 用は true、Translucency 用は false |
| `bSkyVisibility` | `bool` | `false` | Sky Visibility プローブを生成するか。Sky Atmosphere との組み合わせで使用 |

### 使用箇所

- [[ref_lumen_radiance_cache]] — `FUpdateInputs::Configuration` フィールドとして格納
- [[ref_lumen_radiance_cache_hwrt]] — `RenderLumenHardwareRayTracingRadianceCache()` の引数として渡され、Far Field 2 段トレースを制御

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

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `RadianceCacheInputs` | `FRadianceCacheInputs` | プローブ解像度・クリップマップ数・アトラス設定など（[[ref_lumen_radiance_cache_internal]] 参照）|
| `Configuration` | `FRadianceCacheConfiguration` | Far Field / SkyVisibility の有効フラグ |
| `View` | `const FViewInfo&` | カメラビュー情報（クリップマップ中心座標の計算に使用）|
| `ScreenProbeParameters` | `const FScreenProbeParameters*` | Screen Probe との連携パラメータ。nullptr の場合は Screen Probe 連携なし |
| `BRDFProbabilityDensityFunctionSH` | `FRDGBufferSRVRef` | BRDF の重要度分布を SH で表現したバッファ（プローブのトレース方向優先度に使用）|
| `GraphicsMarkUsedRadianceCacheProbes` | `FMarkUsedRadianceCacheProbes` | Graphics パイプラインでプローブを使用済みマークするデリゲート |
| `ComputeMarkUsedRadianceCacheProbes` | `FMarkUsedRadianceCacheProbes` | Compute パイプラインでプローブを使用済みマークするデリゲート |

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — `UpdateRadianceCaches()` の引数として Screen Probe GI 用キャッシュの入力を構築
- [[ref_lumen_translucency_radiance_cache]] — Translucency 用 Radiance Cache の入力として構築
- [[ref_lumen_translucency_volume]] — Translucency Volume 用の Radiance Cache 入力として構築

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

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `RadianceCacheState` | `FRadianceCacheState&` | フレーム間でプローブのアトラス配置を保持する永続状態。`FLumenViewState` 内に存在 |
| `RadianceCacheParameters` | `FRadianceCacheInterpolationParameters&` | シェーダーが Radiance Cache を補間するためのパラメータ。更新後に各パスへ渡される |

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — 更新後の `RadianceCacheParameters` を Screen Probe トレースシェーダーにバインド
- [[ref_lumen_translucency_radiance_cache]] — 透明マテリアルの反射照明用に `RadianceCacheParameters` を使用
- [[ref_lumen_translucency_volume]] — `FLumenTranslucencyGIVolume::RadianceCacheInterpolationParameters` として格納

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

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `RWRadianceProbeIndirectionTexture` | `RWTexture3D<uint>` | プローブ使用状態を書き込む 3D テクスチャ UAV。マークされたセルのビットが立つ |
| `ClipmapCornerTWSAndCellSizeForMark` | `FVector4f[MaxClipmaps]` | 各クリップマップのコーナーワールド座標（XYZ）とセルサイズ（W）。マーク座標変換に使用 |
| `RadianceProbeClipmapResolutionForMark` | `uint32` | クリップマップ 1 辺あたりのプローブ数 |
| `NumRadianceProbeClipmapsForMark` | `uint32` | 有効なクリップマップ数 |
| `InvClipmapFadeSizeForMark` | `float` | クリップマップ境界フェードサイズの逆数（境界付近のプローブもマークするため）|

### 使用箇所

- [[ref_lumen_radiance_cache]] — `FMarkUsedRadianceCacheProbes` デリゲートの引数として渡される
- [[ref_lumen_screen_probe_gather]] — Screen Probe マークパスで使用
- [[ref_lumen_translucency_radiance_cache]] — `FLumenTranslucencyRadianceCacheMarkPassUniformParameters` にインクルード

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

### 使用箇所

- [[ref_lumen_radiance_cache]] — `FUpdateInputs::GraphicsMarkUsedRadianceCacheProbes` / `ComputeMarkUsedRadianceCacheProbes` として格納
- [[ref_lumen_screen_probe_gather]] — Screen Probe がマーク処理をデリゲートに登録
- [[ref_lumen_translucency_radiance_cache]] — Translucency がマーク処理をデリゲートに登録

---

## UpdateRadianceCaches

複数の Radiance Cache を 1 フレームでまとめて更新するエントリポイント。

```cpp
namespace LumenRadianceCache {
    void UpdateRadianceCaches(
        FRDGBuilder& GraphBuilder,
        const FLumenSceneFrameTemporaries& FrameTemporaries,
        const TInlineArray<FUpdateInputs>& InputArray,
        TInlineArray<FUpdateOutputs>& OutputArray,
        const FScene* Scene,
        const FViewFamilyInfo& ViewFamily,
        bool bPropagateGlobalLightingChange,
        ERDGPassFlags ComputePassFlags);
}
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | Surface Cache アトラスバッファ（トレース先テクスチャ）|
| `InputArray` | `const TInlineArray<FUpdateInputs>&` | 更新する各 Radiance Cache の入力設定配列（複数キャッシュを同時更新）|
| `OutputArray` | `TInlineArray<FUpdateOutputs>&` | 各 Radiance Cache の更新結果出力配列 |
| `Scene` | `const FScene*` | シーン（RT-BLAS アクセスなどに使用）|
| `ViewFamily` | `const FViewFamilyInfo&` | ビューファミリー（HW RT 判定など）|
| `bPropagateGlobalLightingChange` | `bool` | スカイライト変更時に全プローブを強制更新するか |
| `ComputePassFlags` | `ERDGPassFlags` | AsyncCompute / Compute の選択フラグ |

### 内部処理フロー

1. **Mark Pass — プローブ使用マーク**
   ```cpp
   // 各入力の FMarkUsedRadianceCacheProbes デリゲートを実行
   // → Screen Probe / Translucency が使用プローブを RWRadianceProbeIndirectionTexture にマーク
   for (auto& Input : InputArray) {
       Input.GraphicsMarkUsedRadianceCacheProbes.Broadcast(GraphBuilder, View, MarkParams);
   }
   ```

2. **Allocate — プローブインデックスのアトラス割り当て**
   ```cpp
   // マークされたプローブを FRadianceCacheState のアトラスに割り当て
   // NumFramesToKeepCachedProbes フレーム以内なら既存インデックスを再利用
   AllocateProbesInAtlas(GraphBuilder, InputArray, SetupOutputArray, ...);
   ```

3. **Trace Budget — 更新プローブの選択**
   ```cpp
   // NumProbesToTraceBudget 個のプローブを古い順・重要度順に選択
   SelectProbesForTracing(GraphBuilder, SetupOutputArray, ProbeTraceDataArray, ...);
   ```

4. **Trace Pass — レイトレース / Surface Cache サンプリング**
   ```cpp
   if (UsesHardwareRayTracing()) {
       // HW RT パス（RT 対応 GPU のみ）
       RenderLumenHardwareRayTracingRadianceCache(GraphBuilder, ...);
   } else {
       // SW RT パス（Surface Cache の FinalLightingAtlas をサンプリング）
       RenderRadianceCacheProbes_SW(GraphBuilder, ...);
   }
   ```

5. **Filter & SH — プローブのフィルタリングと SH 射影**
   ```cpp
   // 空間フィルタリング（SpatialFilterProbes = 1 の場合）
   // → 近傍プローブとの距離重み付き平均でノイズ低減
   FilterRadianceCacheProbes(GraphBuilder, ...);
   // Radiance → SH 係数に射影して FinalRadianceAtlas / FinalIrradianceAtlas に書き込み
   ProjectSHToFinalAtlas(GraphBuilder, ...);
   // OutputArray の RadianceCacheParameters に結果をセット
   GetInterpolationParameters(View, GraphBuilder, RadianceCacheState, RadianceCacheInputs, OutParams);
   ```

### 使用箇所

- [[ref_lumen_screen_probe_gather]] — GI 用 Radiance Cache の更新
- [[ref_lumen_translucency_radiance_cache]] — Translucency Reflection 用 Radiance Cache の更新
- [[ref_lumen_translucency_volume]] — Translucency Volume 用 Radiance Cache の更新
- [[ref_lumen_irradiance_field]] — Irradiance Field Gather モードでの更新

---

## マイナー関数

> [!note]- GetLumenSceneLightingComputePassFlags — Compute パスフラグの取得
>
> ```cpp
> ERDGPassFlags GetLumenSceneLightingComputePassFlags(const FEngineShowFlags& EngineShowFlags);
> ```
>
> **戻り値**: `ERDGPassFlags` — AsyncCompute またはシリアル Compute
>
> **パラメータ**
>
> | 引数 | 型 | 説明 |
> |------|-----|------|
> | `EngineShowFlags` | `const FEngineShowFlags&` | `VisualizeMode` が有効な場合はシリアル Compute を返す（デバッグ用）|
>
> **使用箇所**: [[ref_lumen_scene_lighting]] — Lumen Scene Lighting の Compute パスフラグ決定に使用

> [!note]- UseHitLighting — HW RT Hit Lighting の使用判定
>
> ```cpp
> bool UseHitLighting(const FViewInfo& View, EDiffuseIndirectMethod DiffuseIndirectMethod);
> ```
>
> **戻り値**: `bool` — Hit Lighting（フルマテリアル評価）を使うか
>
> **パラメータ**
>
> | 引数 | 型 | 説明 |
> |------|-----|------|
> | `View` | `const FViewInfo&` | ビュー（CVar 参照）|
> | `DiffuseIndirectMethod` | `EDiffuseIndirectMethod` | GI 方式（Lumen / SSGI 等）|
>
> **使用箇所**: [[ref_lumen_radiance_cache_hwrt]] — HW RT トレース時にフルマテリアル評価を行うか判定
>
> > **補足**: `r.Lumen.HardwareRayTracing.UseHitLighting = 1` かつ DiffuseIndirectMethod が Lumen の場合に `true` を返す

> [!note]- MarkUsedProbesForVisualize — デバッグ可視化用マーク
>
> ```cpp
> extern void MarkUsedProbesForVisualize(
>     FRDGBuilder& GraphBuilder,
>     const FViewInfo& View,
>     const LumenRadianceCache::FRadianceCacheMarkParameters& RadianceCacheMarkParameters,
>     ERDGPassFlags ComputePassFlags);
> ```
>
> **使用箇所**: [[ref_lumen_radiance_cache]] — デバッグ時にプローブを可視化するためにマークパスに追加登録

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
