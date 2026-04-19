# GPU DistanceField ソースマップ

- 対象: DistanceField GPU シェーダー（Mesh SDF Upload + Global SDF Compose + DFAO + DF Shadows）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_distance_field_gpu_overview]]

Mesh SDF（アセット単位）→ Global SDF（クリップマップ合成）→ DFAO（コーントレース AO）+ DF Shadows（コーントレースシャドウ）。
全て Compute Shader ベース、RDG（FRDGBuilder）でパス登録。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/DistanceField*.usf` |
| シェーダー | `Engine/Shaders/Private/DistanceField/*.usf`（Global SDF 専用） |
| CPU | `Renderer/Private/DistanceFieldLightingPost.cpp` / `DistanceFieldAmbientOcclusion.cpp` / `DistanceFieldShadowing.cpp` |

---

## ファイル → シェーダー対応

### Mesh SDF

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `DistanceFieldDownsampling.usf` | `DownsampleMeshSDFCS` | `FDistanceFieldSceneData::UpdateDistanceFieldAtlas()` | [[a_MeshSDF]] |
| `DistanceFieldStreaming.usf` | `ScatterUploadCS` | 同上 | 同 |

### Global SDF

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `DistanceField/GlobalDistanceField.usf` | — | `UpdateGlobalDistanceFieldVolume()` | [[b_GlobalSDF]] |
| `DistanceField/GlobalDistanceFieldCompositeObjects.usf` | `ComposeObjectsIntoClipmap` | 同上 | 同 |
| `DistanceField/GlobalDistanceFieldMip.usf` | `GenerateMipCS` | 同上 | 同 |

### DFAO

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `DistanceFieldObjectCulling.usf` | `CullObjectsForViewCS` | `RenderDistanceFieldLighting()` | [[c_DFAO]] |
| `DistanceFieldScreenGridLighting.usf` | `ComputeDistanceFieldNormalCS` / `AOLevelSetCS` / `ConeTraceCS` | 同上 | 同 |
| `DistanceFieldLightingPost.usf` | `UpsampleBentNormalAO` | 同上 | 同 |

### DF Shadows

| シェーダー | 主要エントリ | CPU 関数 | グループ |
|---------|-----------|---------|---------|
| `DistanceFieldShadowing.usf` | `CullObjectsToShadowFrustumCS` / `RayTraceShadowsCS` | `RayTracedDistanceFieldProjection()` | [[d_DFShadows]] |

---

## 共有ヘッダ

| ヘッダ | 役割 |
|--------|------|
| `DistanceFieldLightingShared.ush` | Object SDF バッファ・アトラス UV 共通 |
| `DistanceFieldAOShared.ush` | DFAO 専用共有型・関数 |
| `DistanceFieldShadowingShared.ush` | DF Shadows 専用共有型・関数 |
| `DistanceField/GlobalDistanceFieldShared.ush` | Global SDF クリップマップサンプリング共通関数 |
| `MeshDistanceFieldCommon.ush` | Mesh SDF 符号付き距離計算共通関数 |

---

## GPU データフロー

```
[Mesh SDF アップロード]            FDistanceFieldSceneData::UpdateDistanceFieldAtlas
  DistanceFieldStreaming.usf:ScatterUploadCS
    → Atlas Texture (Texture3D R16F/R8)

[Global SDF 合成]                   UpdateGlobalDistanceFieldVolume
  GlobalDistanceFieldCompositeObjects.usf:ComposeObjectsIntoClipmap
    → PageAtlasTexture (Texture3D)
  GlobalDistanceFieldMip.usf:GenerateMipCS
    → MipTexture (Texture3D)

[DFAO]                              RenderDistanceFieldLighting
  DistanceFieldObjectCulling.usf:CullObjectsForViewCS
  DistanceFieldScreenGridLighting.usf:ComputeDistanceFieldNormalCS
  DistanceFieldScreenGridLighting.usf:AOLevelSetCS / ConeTraceCS
  DistanceFieldLightingPost.usf:UpsampleBentNormalAO
    → BentNormalAO テクスチャ

[DF Shadows]                        RayTracedDistanceFieldProjection
  DistanceFieldShadowing.usf:CullObjectsToShadowFrustumCS
  DistanceFieldShadowing.usf:RayTraceShadowsCS
    → RayTracedShadowsTexture
```

---

## Details/Reference 一覧

| グループ | 詳細 | リファレンス |
|---------|------|-----------|
| a: Mesh SDF | [[detail_mesh_sdf]] | [[ref_mesh_sdf]] |
| b: Global SDF | [[detail_global_sdf]] | [[ref_global_sdf]] |
| c: DFAO | [[detail_dfao]] | [[ref_dfao]] |
| d: DF Shadows | [[detail_df_shadows]] | [[ref_df_shadows]] |

---

## ue5-dive 起点

- 「Mesh SDF アップロード」 → `DistanceFieldStreaming.usf:ScatterUploadCS`
- 「Global SDF 合成」 → `GlobalDistanceFieldCompositeObjects.usf:ComposeObjectsIntoClipmap`
- 「DFAO コーントレース」 → `DistanceFieldScreenGridLighting.usf:AOLevelSetCS / ConeTraceCS`
- 「DF Shadows」 → `DistanceFieldShadowing.usf:RayTraceShadowsCS`
- 「Global SDF クリップマップサンプル」 → `GlobalDistanceFieldShared.ush`
