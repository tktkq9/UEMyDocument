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

    class FAvoidSelfIntersectionsMode
        : SHADER_PERMUTATION_ENUM_CLASS("AVOID_SELF_INTERSECTIONS_MODE",
            LumenHardwareRayTracing::EAvoidSelfIntersectionsMode);
    class FNaniteRayTracing : SHADER_PERMUTATION_BOOL("NANITE_RAY_TRACING");
    using FPermutationDomain = TShaderPermutationDomain<FAvoidSelfIntersectionsMode, FNaniteRayTracing>;

    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters);
    static ERayTracingPayloadType GetRayTracingPayloadType(const int32 PermutationId);
};
```

### FParameters メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `LumenHardwareRayTracingUniformBuffer` | UB | Lumen RT 全シェーダー共通ユニフォームバッファ（マテリアルデータ・インスタンス情報を含む）|
| `View` | `FViewUniformShaderParameters` | ビュー行列・カメラ情報（バックフェイス判定の法線計算に使用）|
| `NaniteRayTracing` | `FNaniteRayTracingUniformParameters` | Nanite プロキシメッシュ向け RT パラメータ |
| `Scene` | `FSceneUniformParameters` | シーン全体の共通パラメータ |

### パーミュテーション

| パーミュテーション | マクロ | 説明 |
|------------------|--------|------|
| `FAvoidSelfIntersectionsMode` | `AVOID_SELF_INTERSECTIONS_MODE` | 自己交差回避モード（Disabled / Retrace / AHS の 3 値）|
| `FNaniteRayTracing` | `NANITE_RAY_TRACING` | Nanite プロキシメッシュ RT 対応 |

### コンパイル条件

- `RHI_RAYTRACING` が有効なプラットフォーム
- かつ Lumen GI（`r.Lumen.HardwareRayTracing`）または MegaLights（`r.MegaLights`）が有効

### 使用箇所

- [[ref_lumen_hwrt_materials]] — `BuildLumenHardwareRayTracingMaterialPipeline()` で SBT に登録
- [[ref_lumen_hwrt_common]] — `FLumenHardwareRayTracingShaderBase` の利用者（ヒットグループとして接続される）

---

## LumenMinimal ペイロード

Lumen HW RT 専用の最小ペイロード。フルマテリアル評価を行わず、  
Surface Cache への最小情報（ヒット有無・位置）のみを伝達する。

```cpp
// RT ペイロードサイズの登録（16 bytes = float4 × 1）
IMPLEMENT_RT_PAYLOAD_TYPE(ERayTracingPayloadType::LumenMinimal, 16);
```

| 比較項目 | LumenMinimal | DefaultPayload |
|---------|------|------|
| サイズ | 16 bytes | 64+ bytes |
| マテリアル評価 | なし（ヒット有無のみ）| フル評価 |
| 用途 | Lumen シャドウ・GI | ReflectionCapture 等 |
| コスト | 低い | 高い |

### 使用箇所

- [[ref_lumen_scene_direct_lighting_hwrt]] — HW RT Direct Lighting シャドウ判定
- [[ref_lumen_reflections]] — Reflection トレースの初段（Surface Cache モード時）
- [[ref_lumen_screen_probe_hwrt]] — Screen Probe の HW RT トレース

---

## LumenHardwareRayTracing 名前空間（Materials）

```cpp
namespace LumenHardwareRayTracing {
    // ヒットグループ数: 2
    // スロット 0: EAvoidSelfIntersectionsMode::Disabled
    // スロット 1: EAvoidSelfIntersectionsMode::AHS
    constexpr uint32 NumHitGroups = 2;
};
```

### NumHitGroups の意味

```
各プリミティブ × 2 スロットを SBT に確保:
  スロット 0 (Disabled): 自己交差チェックなし → 高速
  スロット 1 (AHS)     : Any Hit Shader で自己交差をフィルタ → 正確

トレース時に InstanceInclusionMask と MissShaderIndex で使い分ける:
  r.Lumen.HardwareRayTracing.AvoidSelfIntersectionsMode = 2 (AHS) の場合はスロット 1 を使用
```

---

## FLumenHardwareRayTracingUniformBufferParameters

Lumen RT 全シェーダーが共有するユニフォームバッファ。  
BindlessResources を通じてマテリアル・インスタンスごとのデータにアクセスするための情報を保持。

```cpp
IMPLEMENT_UNIFORM_BUFFER_STRUCT(
    FLumenHardwareRayTracingUniformBufferParameters,
    "LumenHardwareRayTracingUniformBuffer");
```

### 使用箇所

- [[ref_lumen_hwrt_materials]] — `FLumenHardwareRayTracingMaterialHitGroup::FParameters` に格納
- [[ref_lumen_hwrt_common]] — `FSharedParameters::LumenHardwareRayTracingUniformBuffer` に格納
- すべての Lumen RT シェーダー — `SetLumenHardwareRayTracingSharedParameters()` 経由でバインド

---

## SBT（Shader Binding Table）への登録フロー

```
BuildRayTracingScene() 時:
  │
  ├─ 各プリミティブのマテリアルに対して FLumenHardwareRayTracingMaterialHitGroup を登録
  │   NumHitGroups = 2 → Disabled / AHS の2スロットを確保
  │
  ├─ FHitGroupRootConstants::UserData にプリミティブ固有データインデックスをセット
  │
  └─ SBT に書き込み → GPU が RT ヒット時に自動呼び出し

トレース時:
  RayDesc に InstanceInclusionMask をセット
  LumenMinimal ペイロードでマテリアル評価をスキップ → 高速シャドウ判定
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.HardwareRayTracing.SkipBackFaceHitDistance` | 5.0 | バックフェイスカリングを有効にする距離（cm）。Nanite プロキシジオメトリとのずれ対策 |
| `r.Lumen.HardwareRayTracing.SkipTwoSidedHitDistance` | 1.0 | この距離以内の両面マテリアルヒットをスキップ（フォリッジの自己交差対策）|

---

## 自己交差回避の仕組み

```
EAvoidSelfIntersectionsMode::AHS（Any Hit Shader）使用時:
  レイが自分自身のメッシュに当たった場合を AHS で検出してスキップ
  → より正確だが、AHS コストがかかる
  → r.Lumen.HardwareRayTracing.AvoidSelfIntersectionsMode = 2 で有効

EAvoidSelfIntersectionsMode::Retrace 使用時:
  初回ヒットが自己交差と判定された場合、同じ方向で再トレース
  → シンプルだがコスト増
  → r.Lumen.HardwareRayTracing.AvoidSelfIntersectionsMode = 1 で有効

SkipBackFaceHitDistance = 5.0cm:
  レイ原点から 5cm 以内のバックフェイスヒットをスキップ
  → Nanite の LOD プロキシとラスタライズ GBuffer のずれを補正

SkipTwoSidedHitDistance = 1.0cm:
  両面マテリアルの近距離ヒットをスキップ
  → フォリッジの葉などが自分自身に当たるのを防ぐ
```
