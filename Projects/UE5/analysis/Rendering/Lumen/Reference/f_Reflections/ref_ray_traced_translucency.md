# リファレンス：RayTracedTranslucency.h

- グループ: f - Reflections
- 上位: [[f_lumen_reflections]]
- 関連: [[ref_lumen_front_layer]] | [[ref_lumen_reflection_hwrt]] | [[ref_lumen_hwrt_common]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/RayTracedTranslucency.h`

---

## 概要

透明マテリアルに対するレイトレーシング（**Ray Traced Translucency**）の制御 API を定義するヘッダ。  
透明オブジェクトを通した屈折・反射を HW RT で計算し、  
`LumenFrontLayerTranslucency` より多段レイヤーに対応する高品質モードを提供する。

---

## RayTracedTranslucency 名前空間

### 有効化判定関数

```cpp
namespace RayTracedTranslucency {
    bool IsEnabled(const FViewInfo& View);
    bool UseForceOpaque();
    bool UseRayTracedRefraction(const TArray<FViewInfo>& Views);
    bool AllowTranslucentReflectionInReflections();
}
```

### 有効化判定関数一覧

| 関数名 | 戻り値 | 説明 |
|--------|--------|------|
| `IsEnabled(View)` | `bool` | `r.RayTracing.Translucency` が有効かつ HW RT サポートがある場合 `true` |
| `UseForceOpaque()` | `bool` | `r.RayTracing.Translucency.ForceOpaque = 1` のとき `true`（全透明をOpaque扱い）|
| `UseRayTracedRefraction(Views)` | `bool` | `r.RayTracing.Translucency.Refraction = 1` かつビューが HW RT を要求する場合 `true` |
| `AllowTranslucentReflectionInReflections()` | `bool` | 反射レイ内での透明ネスト（反射→透明→反射）を許可するか |

### 使用箇所

- [[ref_ray_traced_translucency]] — `RenderHardwareRayTracingTranslucency()` がトレース開始前に `IsEnabled()` で有効化確認
- [[ref_lumen_reflection_hwrt]] — `AllowTranslucentReflectionInReflections()` で反射パス内に透明ネストを許可するか判定

---

### 品質制御関数

```cpp
namespace RayTracedTranslucency {
    float    GetPathThroughputThreshold();
    uint32   GetDownsampleFactor(const TArray<FViewInfo>& Views);
    uint32   GetMaxPrimaryHitEvents(const FViewInfo& View);
    uint32   GetMaxSecondaryHitEvents(const FViewInfo& View);
}
```

### 品質制御関数一覧

| 関数名 | 戻り値 | 説明 |
|--------|--------|------|
| `GetPathThroughputThreshold()` | `float` | パス打ち切り輝度閾値。累積透過率がこれ以下になったらレイ追跡を終了（デフォルト: 0.01 = 1%）|
| `GetDownsampleFactor(Views)` | `uint32` | 解像度ダウンサンプル係数。`r.RayTracing.Translucency.DownsampleFactor`（デフォルト: 1）|
| `GetMaxPrimaryHitEvents(View)` | `uint32` | 一次ヒットの最大数（透明レイヤーを何枚まで通過するか）。`r.RayTracing.Translucency.MaxPrimaryHitCount`（デフォルト: 8）|
| `GetMaxSecondaryHitEvents(View)` | `uint32` | 二次ヒットの最大数（反射・屈折後の追加ヒット）。`r.RayTracing.Translucency.MaxSecondaryHitCount`（デフォルト: 2）|

### 使用箇所

- [[ref_ray_traced_translucency]] — `RenderHardwareRayTracingTranslucency()` がシェーダーパラメータ設定時に全品質関数を参照
- [[ref_lumen_reflection_hwrt]] — マルチバウンス反射で `GetMaxSecondaryHitEvents()` を透明ヒット後の二次レイ本数上限として参照

---

## TraceTranslucency

```cpp
extern void TraceTranslucency(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    ...);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（SDF / TLAS アクセス）|
| `View` | `const FViewInfo&` | カメラビュー |
| `SceneTextures` | `const FSceneTextures&` | HZB・GBuffer テクスチャ群 |
| `...` | — | 実装ファイル側で拡張（TracingParameters 等）|

### 使用箇所

- SW パス（スクリーンスペース + SDF）での透明トレース。`RayTracedTranslucency::IsEnabled()` が `false` の場合のフォールバック

---

## RenderHardwareRayTracingTranslucency

```cpp
extern void RenderHardwareRayTracingTranslucency(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    const FLumenCardTracingParameters& TracingParameters,
    ERDGPassFlags ComputePassFlags,
    ...);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（TLAS / Far Field TLAS アクセス）|
| `View` | `const FViewInfo&` | カメラビュー |
| `SceneTextures` | `const FSceneTextures&` | SceneColor 等の書き込み先テクスチャ |
| `TracingParameters` | `const FLumenCardTracingParameters&` | Surface Cache アトラス・トレース共通パラメータ |
| `ComputePassFlags` | `ERDGPassFlags` | AsyncCompute / Compute の選択フラグ |
| `...` | — | DownsampleFactor / MaxPrimaryHitEvents 等の品質パラメータ |

### 使用箇所

- [[ref_ray_traced_translucency]] — `RayTracedTranslucency::IsEnabled()` が `true` の場合に呼ばれる HW RT メインパス

### 内部処理フロー

1. **品質パラメータの取得**
   ```cpp
   uint32 MaxPrimary = GetMaxPrimaryHitEvents(View);
   uint32 MaxSecondary = GetMaxSecondaryHitEvents(View);
   float  ThroughputThreshold = GetPathThroughputThreshold();
   ```

2. **一次レイ発射**
   ```cpp
   // 透明オブジェクトのサーフェスにヒットするまでトレース
   // MaxPrimaryHitEvents 回まで透明面を通過
   DispatchTranslucencyRayGen(GraphBuilder, MaxPrimary, ...);
   ```

3. **屈折計算**
   ```cpp
   if (UseRayTracedRefraction(Views)) {
       // ヒット点の法線・IOR から屈折ベクトルを計算して方向変更
       ComputeRefractionDirection(HitNormal, IOR, ...);
   }
   ```

4. **透明面の反射**
   ```cpp
   // 各透明ヒット点で反射レイを発射（MaxSecondaryHitEvents 回）
   // Surface Cache または Hit Lighting でラジアンスを取得
   for (int i = 0; i < MaxSecondary; ++i) {
       DispatchSecondaryReflectionRay(GraphBuilder, ...);
   }
   ```

5. **パススルー制御**
   ```cpp
   // Throughput（累積透過率）が閾値を下回ったらレイ打ち切り
   if (Throughput < ThroughputThreshold) { break; }
   ```

6. **最終合成**
   ```cpp
   // 最終カラーを SceneColor に加算合成
   CompositeTranslucencyToSceneColor(GraphBuilder, SceneTextures, ...);
   ```

---

## LumenFrontLayerTranslucency との差異

| 比較項目 | FrontLayer（簡易）| RayTracedTranslucency（高品質）|
|---------|---------|---------|
| 対応レイヤー数 | 1層（最前面のみ）| 多層（MaxPrimaryHitEvents まで）|
| 屈折対応 | なし | UseRayTracedRefraction = true 時に対応 |
| 反射内での利用 | 基本的に不可 | AllowTranslucentReflectionInReflections で制御 |
| GPU コスト | 低 | 高（多段トレース）|
| 用途 | インゲーム（実時間）| ハイエンド / 映像制作 |

---

## 透明マテリアルのレイトレース処理フロー

```
RenderHardwareRayTracingTranslucency()
  │
  ├─ [品質パラメータ取得]
  │   GetMaxPrimaryHitEvents() / GetMaxSecondaryHitEvents() / GetPathThroughputThreshold()
  │
  ├─ [一次レイ発射]
  │   透明オブジェクトのサーフェスにヒットするまでトレース
  │   MaxPrimaryHitEvents 回まで透明面を通過
  │
  ├─ [屈折計算]
  │   UseRayTracedRefraction() = true の場合:
  │   ヒット点の法線・IOR から屈折ベクトルを計算して方向変更
  │
  ├─ [透明面の反射]
  │   各透明ヒット点で反射レイを発射（MaxSecondaryHitEvents 回）
  │   → Surface Cache または Hit Lighting でラジアンスを取得
  │
  ├─ [パススルー制御]
  │   Throughput（累積透過率）が GetPathThroughputThreshold() を下回ったら打ち切り
  │
  └─ 最終カラーを SceneColor に合成
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.RayTracing.Translucency` | 0 | Ray Traced Translucency の有効/無効 |
| `r.RayTracing.Translucency.Refraction` | 1 | 屈折トレースの有効/無効 |
| `r.RayTracing.Translucency.MaxRefractionRays` | 3 | 屈折レイの最大本数 |
| `r.RayTracing.Translucency.PrimaryRayFlags` | 0 | 一次レイのフラグ設定 |
| `r.RayTracing.Translucency.PathThroughputThreshold` | 0.01 | パス打ち切り輝度閾値（1%）|
| `r.RayTracing.Translucency.DownsampleFactor` | 1 | 解像度ダウンサンプル係数 |
| `r.RayTracing.Translucency.MaxRoughness` | 0.3 | RT を適用する最大ラフネス |
| `r.RayTracing.Translucency.ForceOpaque` | 0 | 全透明をOpaque扱いにする（デバッグ用）|
| `r.RayTracing.Translucency.MaxPrimaryHitCount` | 8 | 一次ヒットの最大数 |
| `r.RayTracing.Translucency.MaxSecondaryHitCount` | 2 | 二次ヒットの最大数 |
