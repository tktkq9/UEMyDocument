# GPU RayTracing ソースマップ

- 対象: HW RT GPU シェーダー（TLAS/BLAS + Shadow/AO/SkyLight/Reflection/Translucency + 組み込み）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_raytracing_gpu_overview]]

`RHI_RAYTRACING` ガードで有効化。DXR/Vulkan Ray Tracing 対応 GPU 限定。
RayGen/ClosestHit/AnyHit/Miss + Inline（CS）の各エントリタイプを使い分ける。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/RayTracing/*.usf` |
| シェーダー共通 | `Engine/Shaders/Private/RayTracing/RayTracingCommon.ush` ほか |
| CPU | `Renderer/Private/RayTracing/*.cpp`（概要: [[07_raytracing_overview]]） |

---

## ファイル → シェーダー対応

### Core（TLAS/BLAS 構築 + 組み込み）

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `RayTracingInstanceBufferUtil.usf` | `BuildRayTracingInstanceBufferCS()` | `FRayTracingScene::Update()` | BLAS インスタンスを GPU バッファへ | [[detail_core]] |
| `RayTracingDynamicMesh.usf` | `RayTracingDynamicGeometryConverterCS()` | `RayTracingDynamicGeometryUpdateManager` | 動的ジオメトリ BLAS 更新 | 同 |
| `RayTracingBuiltInShaders.usf` | `OcclusionMainRGS` / `IntersectionMainRGS` / `DefaultMainCHS` | 組み込み | テスト/デバッグ/デフォルトフォールバック | 同 |

### Shadow（Occlusion）

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `RayTracingOcclusionRGS.usf` | `Occlusion`（RGS / Inline CS） | `RenderRayTracingShadows()` | ライト別 Occlusion レイ → OcclusionMask | [[detail_shadow]] |
| `RayTracingMaterialDefaultHitShaders.usf` | `DefaultMainCHS` / `DefaultMainAHS` | 同上 | デフォルトヒットシェーダー | 同 |

### AO / SkyLight

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `RayTracingAmbientOcclusionRGS.usf` | `AmbientOcclusionRGS` | `RenderRayTracingAmbientOcclusion()` | コサインサンプリング AO | [[detail_ao]] |
| `RayTracingSkyLightRGS.usf` | `SkyLightRGS` | `RenderRayTracingSkyLight()` | Sky Occlusion | 同 |
| `GenerateSkyLightVisibilityRaysCS.usf` | `MainCS` | 同上 | 可視性レイ事前生成 | 同 |

### Reflection / Translucency

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `RayTracingPrimaryRays.usf` | `RayTracingPrimaryRaysRGS` | `RenderRayTracingReflections()` / `RenderRayTracingTranslucency()` | 1 次レイ（反射 + 半透明、Translucent パーミュテーション） | [[detail_reflection]] / [[detail_translucency]] |
| `RayTracingMaterialHitShaders.usf` | `MaterialCHS` / `MaterialAHS` | `SetRayTracingShaderBindings()` | マテリアル評価ヒットシェーダー | 同 |
| `RayTracingLightingMS.usf` | `RayTracingLightingMS` | 同上 | Miss シェーダー（ライティング） | 同 |
| `RayTracingDeferredMaterials.usf` | 複数 | — | 遅延マテリアル評価 | 同 |

---

## GPU データフロー

```
[A] TLAS/BLAS 構築                  RayTracing::OnRenderBegin
    RayTracingInstanceBufferUtil.usf:BuildRayTracingInstanceBufferCS
    RayTracingDynamicMesh.usf:RayTracingDynamicGeometryConverterCS
    → FRayTracingScene::Update/Build（RHI 側で TLAS ビルド）

[B] Shadow                          RenderRayTracingShadows × 光源数
    RayTracingOcclusionRGS.usf:Occlusion（RGS / Inline）
    → OcclusionMask → IScreenSpaceDenoiser

[C] AO                              RenderRayTracingAmbientOcclusion
    RayTracingAmbientOcclusionRGS.usf:AmbientOcclusionRGS

[D] SkyLight                        RenderRayTracingSkyLight
    RayTracingSkyLightRGS.usf:SkyLightRGS
    GenerateSkyLightVisibilityRaysCS.usf:MainCS

[E] Reflection                      RenderRayTracingReflections
    RayTracingPrimaryRays.usf:RayTracingPrimaryRaysRGS
    RayTracingMaterialHitShaders.usf:MaterialCHS/MaterialAHS

[F] Translucency                    RenderRayTracingTranslucency
    RayTracingPrimaryRays.usf（Translucent パーミュテーション）
    RayTracingLightingMS.usf:RayTracingLightingMS

[Lumen HW RT]   → 同じ TLAS を LumenHardwareRayTracing から参照
[MegaLights RT] → 同じ TLAS を MegaLightsHardwareRayTracing から参照
```

---

## シェーダーエントリタイプ

| タイプ | マクロ |
|--------|--------|
| Ray Generation | `RAY_TRACING_ENTRY_RAYGEN` |
| Closest Hit | `RAY_TRACING_ENTRY_CLOSEST_HIT` |
| Any Hit | `RAY_TRACING_ENTRY_ANY_HIT` |
| Miss | `RAY_TRACING_ENTRY_MISS` |
| Inline（CS） | `RAY_TRACING_ENTRY_RAYGEN_OR_INLINE` |

---

## 主要ペイロード型

| 型 | ファイル | 用途 |
|-----|--------|------|
| `FDefaultPayload` | `RayTracingCommon.ush` | 最小ペイロード（Occlusion） |
| `FPackedMaterialClosestHitPayload` | 同 | マテリアル結果（圧縮） |
| `FMaterialClosestHitPayload` | 同 | マテリアル結果（展開） |

---

## 共有ヘッダ

| ヘッダ | 内容 |
|--------|------|
| `RayTracingCommon.ush` | ペイロード型・共通関数 |
| `RayTracingDeferredShadingCommon.ush` | GBuffer 参照補助 |
| `RayTracingHitGroupCommon.ush` | ヒットグループ共通 |
| `RayTracingCalcInterpolants.ush` | 補間計算 |
| `TraceRayInline.ush` | Inline RT |
| `RayGenUtils.ush` | RayGen 共通 |

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_core]] / [[detail_shadow]] / [[detail_ao]] / [[detail_reflection]] / [[detail_translucency]] |
| Reference | [[ref_core]] / [[ref_shadow]] / [[ref_ao]] / [[ref_reflection]] / [[ref_translucency]] |

---

## ue5-dive 起点

- 「TLAS 構築 CS」 → `RayTracingInstanceBufferUtil.usf:BuildRayTracingInstanceBufferCS`
- 「RT Shadow」 → `RayTracingOcclusionRGS.usf:Occlusion`
- 「RT Reflection」 → `RayTracingPrimaryRays.usf:RayTracingPrimaryRaysRGS`
- 「マテリアルヒットシェーダー」 → `RayTracingMaterialHitShaders.usf:MaterialCHS/MaterialAHS`
- 「Inline RT（CS）」 → `TraceRayInline.ush`
