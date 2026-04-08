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
    // Ray Traced Translucency が有効かどうか
    bool IsEnabled(const FViewInfo& View);

    // 不透明フラグ強制を使うか（デバッグ・負荷削減用）
    bool UseForceOpaque();

    // 屈折（Refraction）に RT を使うか
    bool UseRayTracedRefraction(const TArray<FViewInfo>& Views);

    // 透明マテリアルの反射内で RT Translucency を許可するか
    // （反射→透明→反射 の多重ネストを許可するか）
    bool AllowTranslucentReflectionInReflections();
}
```

### 品質制御関数

```cpp
namespace RayTracedTranslucency {
    // パススルー輝度の閾値（これ以下のレイは追跡を打ち切る）
    float GetPathThroughputThreshold();

    // 解像度ダウンサンプル係数
    uint32 GetDownsampleFactor(const TArray<FViewInfo>& Views);

    // 一次ヒットイベントの最大数（透明レイヤーを何枚まで通過するか）
    uint32 GetMaxPrimaryHitEvents(const FViewInfo& View);

    // 二次ヒットイベントの最大数（反射・屈折後の追加ヒット）
    uint32 GetMaxSecondaryHitEvents(const FViewInfo& View);
}
```

---

## 主要関数（extern 宣言）

```cpp
// SW パス（スクリーンスペース + SDF）での透明トレース
extern void TraceTranslucency(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    ...);

// HW RT パスでの透明トレース
extern void RenderHardwareRayTracingTranslucency(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    const FLumenCardTracingParameters& TracingParameters,
    ERDGPassFlags ComputePassFlags,
    ...);
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
