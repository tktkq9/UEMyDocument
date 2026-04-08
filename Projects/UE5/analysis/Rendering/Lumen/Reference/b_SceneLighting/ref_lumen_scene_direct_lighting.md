# リファレンス：LumenSceneDirectLighting.cpp / LumenSceneDirectLightingStochastic.inl

- グループ: b - Scene Lighting
- 上位: [[b_lumen_scene_lighting]]
- 関連: [[ref_lumen_scene_lighting]] | [[ref_lumen_scene_direct_lighting_hwrt]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneDirectLighting.cpp`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneDirectLightingStochastic.inl`

---

## 概要

Surface Cache の **Direct Lighting**（直接照明）計算を担うファイル。  
シーン内の全ライトを収集し、Card タイルごとに光源をカリング・シャドウを解決して  
Direct Lighting アトラスに書き込む。  
標準パス（決定論的）と確率的ライティングパス（`Stochastic.inl`）の2系統を持つ。

---

## FLumenGatheredLight

Lumen Direct Lighting で処理する**ライト1灯分のデータ**をまとめたクラス。

```cpp
class FLumenGatheredLight {
public:
    const FLightSceneInfo* LightSceneInfo = nullptr;         // ライトのシーン情報
    const FMaterialRenderProxy* LightFunctionMaterialProxy;  // ライト関数マテリアル
    uint32 LightIndex = 0;           // GatheredLights 配列内インデックス
    ELumenLightType Type;            // Directional / Point / Spot / Rect
    bool bHasShadows = false;        // ダイナミックシャドウを落とすか
    bool bMayCastCloudTransmittance; // 雲シャドウの可能性があるか
    bool bNeedsShadowMask = false;   // シャドウマスクが必要か
    bool bBatchedShadowsEligible;    // バッチシャドウ処理が可能か
    FString Name;                    // デバッグ用ライト名

    // ビューオリジンごとの Deferred Light UB（マルチビュー対応）
    TArray<TUniformBufferRef<FDeferredLightUniformStruct>, TInlineAllocator<4>> DeferredLightUniformBuffers;

    bool NeedsShadowMask() const;
    bool CanUseBatchedShadows() const;
};
```

---

## FLumenLightTileScatterParameters

ライトタイルの Indirect Dispatch に使うシェーダーパラメータ構造体。

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenLightTileScatterParameters, )
    RDG_BUFFER_ACCESS(DrawIndirectArgs, ERHIAccess::IndirectArgs)
    RDG_BUFFER_ACCESS(DispatchIndirectArgs, ERHIAccess::IndirectArgs)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>,  LightTileAllocator)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint2>, LightTiles)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<uint>,  LightTileOffsetsPerLight)
    SHADER_PARAMETER(int32, bUseLightTilesPerLightType)
END_SHADER_PARAMETER_STRUCT()
```

---

## LumenSceneDirectLighting 名前空間

### シャドウレイバイアス関数

各トレースメソッドごとに独立したバイアス値を返す。  
`FMath::Max(..., 0.0f)` でゼロ未満を防ぐ。

| 関数 | CVar | デフォルト |
|-----|------|----------|
| `GetMeshSDFShadowRayBias()` | `r.LumenScene.DirectLighting.MeshSDF.ShadowRayBias` | 2.0 |
| `GetHeightfieldShadowRayBias()` | `r.LumenScene.DirectLighting.Heightfield.ShadowRayBias` | 2.0 |
| `GetGlobalSDFShadowRayBias()` | `r.LumenScene.DirectLighting.GlobalSDF.ShadowRayBias` | 1.0 |
| `GetHardwareRayTracingShadowRayBias()` | `r.LumenScene.DirectLighting.HardwareRayTracing.ShadowRayBias` | 1.0 |

### 判定関数

```cpp
// バッチシャドウ: r.LumenScene.DirectLighting.BatchShadows == 2 でライトタイプごとにタイル分割
bool UseLightTilesPerLightType();

// 確率的ライティングを使うか（ScreenProbe との連携）
bool UseStochasticLighting(const FSceneViewFamily& ViewFamily);

// HW RT Direct Lighting の有効判定
// RHI_RAYTRACING && Lumen::UseHardwareRayTracing() && CVar != 0
bool Lumen::UseHardwareRayTracedDirectLighting(const FSceneViewFamily&);

// Far Field を Direct Lighting で使うか
bool UseFarField(const FSceneViewFamily& ViewFamily);

// 全面を両面としてトレースするか（高速化だがラスタライズとの乖離が生じる）
bool IsForceTwoSided();
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.DirectLighting` | 1 | Surface Cache の Direct Lighting の有効/無効 |
| `r.LumenScene.DirectLighting.MaxLightsPerTile` | 8 | タイルあたりの最大ライト数（4/8/16/32）|
| `r.LumenScene.DirectLighting.CullToTileDepthRange` | 1 | タイルの深度範囲でライトをカリングするか |
| `r.LumenScene.DirectLighting.OffscreenShadowing.TraceMeshSDFs` | 1 | オフスクリーンシャドウに Mesh SDF を使うか（0 = Global SDF）|
| `r.LumenScene.DirectLighting.OffscreenShadowingTraceStepFactor` | 5.0 | オフスクリーンシャドウのトレースステップ係数 |
| `r.LumenScene.DirectLighting.CloudTransmittance` | 1 | 雲シャドウのサンプリングを行うか |
| `r.LumenScene.DirectLighting.BatchShadows` | 2 | シャドウパスのバッチ処理（0=無効, 1=タイプ共通, 2=タイプ別）|
| `r.LumenScene.DirectLighting.HardwareRayTracing.AdaptiveShadowTracing` | 1 | 前フレームで均一シャドウだったタイルのレイ数を削減 |

---

## Direct Lighting パイプライン全体フロー

```
RenderLumenSceneLighting()
  │
  └─ RenderDirectLighting(CardUpdateContext)
        │
        ├─ GatherLights()          ← シーン内の有効ライトを FLumenGatheredLight に収集
        │
        ├─ BuildLightTiles()       ← Card タイルごとにどのライトが影響するか計算
        │   ├─ CullToTileDepthRange → タイル深度範囲でタイトなカリング
        │   └─ LightTiles バッファに書き込み
        │
        ├─ [UseStochasticLighting == true]
        │   └─ RenderDirectLightingStochastic()  ← Stochastic.inl 側
        │         → 1 Card タイルあたり N サンプルでランダムライト選択
        │         → CompactedLightSampleData に書き込み
        │
        ├─ [UseStochasticLighting == false]
        │   └─ RenderDirectLightingStandard()
        │         → ライトタイプ別にシャドウレイを発射
        │         → UseHardwareRayTracedDirectLighting() ? HW RT : SDF
        │
        └─ CompositeDirectLighting()  → Direct Lighting アトラスに合成
```

---

## バッチ処理可能ライトタイプ

```cpp
// Point / Spot / Rect の 3 種がバッチ対象
// Directional は雲シャドウ・ライト関数の可能性があるため個別処理
constexpr uint32 NumBatchableLightTypes = 3;
```
