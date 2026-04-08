# リファレンス：LumenSceneDirectLightingHardwareRayTracing.cpp

- グループ: b - Scene Lighting
- 上位: [[b_lumen_scene_lighting]]
- 関連: [[ref_lumen_scene_direct_lighting]] | [[ref_lumen_hwrt_common]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneDirectLightingHardwareRayTracing.cpp`

---

## 概要

Surface Cache Direct Lighting の **Hardware Ray Tracing（HW RT）シャドウ**実装ファイル。  
SDF ベースのシャドウと比較して、より正確なシャドウを高解像度 Card に提供する。  
`#if RHI_RAYTRACING` ブロック内に実装されており、RT 対応 GPU でのみコンパイルされる。

---

## Lumen 名前空間（HW RT 判定）

```cpp
namespace Lumen {
    // HW RT Direct Lighting を使用するかどうかの判定
    // 条件: RHI_RAYTRACING && IsRayTracingEnabled()
    //        && Lumen::UseHardwareRayTracing(ViewFamily)
    //        && r.LumenScene.DirectLighting.HardwareRayTracing != 0
    bool UseHardwareRayTracedDirectLighting(const FSceneViewFamily& ViewFamily);
}
```

---

## FLumenSceneDebugHardwareRayTracing

HW RT を使ったデバッグ用シャドウトレースシェーダー。  
`TraceLumenHardwareRayTracedDebug()` から呼ばれ、デバッグ描画に使用される。

```cpp
class FLumenSceneDebugHardwareRayTracing : public FLumenHardwareRayTracingShaderBase {
    DECLARE_LUMEN_RAYTRACING_SHADER(FLumenSceneDebugHardwareRayTracing)

    using FPermutationDomain = TShaderPermutationDomain<
        FLumenHardwareRayTracingShaderBase::FBasePermutationDomain>;

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_STRUCT_INCLUDE(FLumenHardwareRayTracingShaderBase::FSharedParameters, SharedParameters)
        SHADER_PARAMETER_STRUCT_INCLUDE(ShaderPrint::FShaderParameters, ShaderPrintUniformBuffer)
        SHADER_PARAMETER(float, ResolutionScale)
        SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>, RWDebugData)
    END_SHADER_PARAMETER_STRUCT()

    static bool ShouldCompilePermutation(...);
    static void ModifyCompilationEnvironment(...);

    // ペイロードタイプ: LumenMinimal（シャドウ判定のみ）
    static ERayTracingPayloadType GetRayTracingPayloadType(const int32 PermutationId);
};
```

---

## LumenSceneDirectLighting 名前空間（HW RT 関連）

```cpp
namespace LumenSceneDirectLighting {
    // Far Field を HW RT Direct Lighting で使うか
    // 条件: Lumen::UseFarField() && r.LumenScene.DirectLighting.HardwareRayTracing.FarField != 0
    bool UseFarField(const FSceneViewFamily& ViewFamily);

    // 全メッシュを両面強制するか（高速化、ただしラスタライズとの差異が生じる）
    bool IsForceTwoSided();
}
```

---

## エントリポイント関数

```cpp
// HW RT シャドウトレースのメインエントリ
// Card タイルごとに RT シャドウレイを発射し、シャドウマスクバッファに書き込む
void TraceLumenHardwareRayTracedDirectLightingShadows(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    const FLumenCardUpdateContext& CardUpdateContext,
    const FLumenCardTileUpdateContext& CardTileUpdateContext,
    const TArray<FLumenGatheredLight>& GatheredLights,
    FRDGBufferRef ShadowMaskTiles);

// デバッグ用: RT シャドウトレースの中間データを SRV として返す
FRDGBufferSRVRef TraceLumenHardwareRayTracedDebug(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    float ResolutionScale);
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.DirectLighting.HardwareRayTracing` | 1 | HW RT Direct Lighting の有効/無効 |
| `r.LumenScene.DirectLighting.HardwareRayTracing.ForceTwoSided` | 0 | 全メッシュを両面扱いにするか |
| `r.LumenScene.DirectLighting.HardwareRayTracing.EndBias` | 1.0 | プロキシジオメトリの自己遮蔽防止バイアス |
| `r.LumenScene.DirectLighting.HardwareRayTracing.FarField` | 1 | Far Field を HW RT シャドウで使うか |
| `r.LumenScene.DirectLighting.HardwareRayTracing.HeightfieldProjectionBias` | 0 | Heightfield 投影バイアスを適用するか |
| `r.LumenScene.DirectLighting.HardwareRayTracing.HeightfieldProjectionBiasSearchRadius` | 256 | 投影バイアスの探索半径（大きいほど負荷増）|
| `r.LumenScene.DirectLighting.HardwareRayTracing.AdaptiveShadowTracing` | 1 | 均一シャドウタイルのレイ数を削減するか |

---

## HW RT シャドウの仕組み

```
TraceLumenHardwareRayTracedDirectLightingShadows()
  │
  ├─ ライトごとにタイルをカリング（LightTiles バッファ参照）
  │
  ├─ 各 Card タイル × 各ライトに対してシャドウレイを Dispatch
  │   ├─ ペイロード: ERayTracingPayloadType::LumenMinimal
  │   │   （位置・法線のみ、マテリアル評価なし → 高速）
  │   ├─ AdaptiveShadowTracing: 前フレームで均一シャドウ → レイ削減
  │   └─ ForceTwoSided: false → 裏面はシャドウを落とさない
  │
  ├─ Far Field 有効時: 近距離 BLAS の miss 後に Far Field をトレース
  │
  └─ ShadowMaskTiles バッファに 0/1 を書き込み
        → CompositeDirectLighting() でアトラスに合成
```

---

## SDF シャドウとの比較

| 比較項目 | SDF シャドウ | HW RT シャドウ |
|---------|------------|--------------|
| 精度 | 低〜中（SDF の近似誤差）| 高（ジオメトリに対して正確）|
| 速度 | 速い | 遅い（RT 対応ハードが必要）|
| 有効条件 | 常に利用可 | RT 対応 GPU のみ |
| バイアス調整 | MeshSDF / GlobalSDF 別に設定 | HardwareRayTracing.ShadowRayBias |
