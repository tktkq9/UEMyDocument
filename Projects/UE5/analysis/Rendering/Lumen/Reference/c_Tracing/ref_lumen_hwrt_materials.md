# リファレンス：LumenHardwareRayTracingMaterials.cpp

- グループ: c - Tracing
- 上位: [[c_lumen_tracing]]
- 関連: [[ref_lumen_hwrt_common]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenHardwareRayTracingMaterials.cpp`

---

## 概要

Lumen HW RT で使われる **マテリアルヒットグループシェーダー**を実装するファイル。  
`#if RHI_RAYTRACING` ブロック内にのみ実装され、RT 対応 GPU 専用コードとなる。  
LumenMinimal ペイロードを使い、マテリアル評価を最小限に抑えた高速なシャドウ判定を実現する。

---

## FLumenHardwareRayTracingMaterialHitGroup

Lumen RT の**ヒットグループシェーダー**。  
SBT（Shader Binding Table）に登録され、RT レイがメッシュに交差したときに呼ばれる。

```cpp
class FLumenHardwareRayTracingMaterialHitGroup : public FGlobalShader {
    DECLARE_GLOBAL_SHADER(FLumenHardwareRayTracingMaterialHitGroup)
    SHADER_USE_ROOT_PARAMETER_STRUCT(FLumenHardwareRayTracingMaterialHitGroup, FGlobalShader)

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_STRUCT_REF(FLumenHardwareRayTracingUniformBufferParameters, LumenHardwareRayTracingUniformBuffer)
        SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
        SHADER_PARAMETER_STRUCT_REF(FNaniteRayTracingUniformParameters, NaniteRayTracing)
        SHADER_PARAMETER_STRUCT_REF(FSceneUniformParameters, Scene)
    END_SHADER_PARAMETER_STRUCT()

    // パーミュテーション
    class FAvoidSelfIntersectionsMode
        : SHADER_PERMUTATION_ENUM_CLASS("AVOID_SELF_INTERSECTIONS_MODE",
            LumenHardwareRayTracing::EAvoidSelfIntersectionsMode);
    class FNaniteRayTracing : SHADER_PERMUTATION_BOOL("NANITE_RAY_TRACING");
    using FPermutationDomain = TShaderPermutationDomain<FAvoidSelfIntersectionsMode, FNaniteRayTracing>;

    // コンパイル条件: RT サポート && (Lumen GI または MegaLights)
    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters);

    // ペイロードタイプ: LumenMinimal（16 bytes — 最小限のヒット情報のみ）
    static ERayTracingPayloadType GetRayTracingPayloadType(const int32 PermutationId);
};
```

---

## LumenMinimal ペイロード

Lumen HW RT 専用の最小ペイロード。フルマテリアル評価を行わず、  
Surface Cache への最小情報（ヒット有無・位置）のみを伝達する。

```cpp
// RT ペイロードサイズの登録（16 bytes = float4 × 1）
IMPLEMENT_RT_PAYLOAD_TYPE(ERayTracingPayloadType::LumenMinimal, 16);
```

---

## LumenHardwareRayTracing 名前空間（Materials）

```cpp
namespace LumenHardwareRayTracing {
    // ヒットグループ数: 2
    // 0: EAvoidSelfIntersectionsMode::Disabled
    // 1: EAvoidSelfIntersectionsMode::AHS
    constexpr uint32 NumHitGroups = 2;
};
```

---

## FLumenHardwareRayTracingUniformBufferParameters

Lumen RT 全シェーダーが共有するユニフォームバッファ。

```cpp
IMPLEMENT_UNIFORM_BUFFER_STRUCT(
    FLumenHardwareRayTracingUniformBufferParameters,
    "LumenHardwareRayTracingUniformBuffer");
```

---

## 主要 CVar（LumenHardwareRayTracingMaterials.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.HardwareRayTracing.SkipBackFaceHitDistance` | 5.0 | バックフェイスカリングを有効にする距離（Nanite プロキシジオメトリとの不一致対策）|
| `r.Lumen.HardwareRayTracing.SkipTwoSidedHitDistance` | 1.0 | SkipBackFaceHitDistance 有効時、この距離以内の両面マテリアルヒットをスキップ（フォリッジの自己交差対策）|

---

## 自己交差回避の仕組み

```
EAvoidSelfIntersectionsMode::AHS（Any Hit Shader）使用時:
  レイが自分自身のメッシュに当たった場合を AHS で検出してスキップ
  → より正確だが、AHS コストがかかる

EAvoidSelfIntersectionsMode::Retrace 使用時:
  初回ヒットが自己交差と判定された場合、同じ方向で再トレース
  → シンプルだがコスト増

SkipBackFaceHitDistance:
  レイ原点から 5.0cm 以内のバックフェイスヒットをスキップ
  → Nanite の LOD プロキシとラスタライズ GBuffer のずれを補正
```

---

## SBT（Shader Binding Table）への登録

```
BuildRayTracingScene() 時:
  各プリミティブのマテリアルに対して FLumenHardwareRayTracingMaterialHitGroup を登録
  NumHitGroups = 2 → Disabled / AHS の2スロットを確保

トレース時:
  RayDesc に InstanceInclusionMask をセットしてヒットグループを選択
  LumenMinimal ペイロードでマテリアル評価をスキップ → 高速シャドウ判定
```
