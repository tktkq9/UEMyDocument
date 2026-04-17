# GPU: DistanceField シェーダー全体概要

- 上位: [[01_gpu_overview]]
- CPU 対応: [[10_distance_field_overview]]

---

## 概要

Distance Field（SDF）GPU シェーダーは、Mesh SDF のアップロードから Global SDF 合成、  
DFAO コーントレース、DF Shadows コーントレースまでの一連のパスを担う。  
いずれも **Compute Shader** ベースで、RDG（FRDGBuilder）経由でパス登録される。

---

## GPU シェーダーグループ対応表

| グループ | シェーダーファイル | 主要 CS エントリ | 用途 |
|---------|-----------------|---------------|------|
| **a: Mesh SDF** | `DistanceFieldDownsampling.usf`<br>`DistanceFieldStreaming.usf` | `DownsampleMeshSDFCS`<br>`ScatterUploadCS` | Mesh SDF の Atlas アップロード・Mip 生成 |
| **b: Global SDF** | `DistanceField/GlobalDistanceField.usf`<br>`DistanceField/GlobalDistanceFieldCompositeObjects.usf`<br>`DistanceField/GlobalDistanceFieldMip.usf` | `ComposeObjectsIntoClipmap`<br>`GenerateMipCS` | Mesh SDF → クリップマップへの合成・Mip 生成 |
| **c: DFAO** | `DistanceFieldObjectCulling.usf`<br>`DistanceFieldScreenGridLighting.usf`<br>`DistanceFieldLightingPost.usf` | `CullObjectsForViewCS`<br>`ComputeDistanceFieldNormalCS`<br>`AOLevelSetCS` | Global SDF コーントレース AO・Bent Normal 生成 |
| **d: DF Shadows** | `DistanceFieldShadowing.usf` | `CullObjectsToShadowFrustumCS`<br>`RayTraceShadowsCS` | Mesh SDF コーントレースによるソフトシャドウ |

---

## シェーダー間の共有ヘッダ

| ヘッダ | 役割 |
|--------|------|
| `DistanceFieldLightingShared.ush` | Object SDF バッファ・アトラス UV アクセス共通定義 |
| `DistanceFieldAOShared.ush` | DFAO 専用の共有型・関数 |
| `DistanceFieldShadowingShared.ush` | DF Shadows 専用の共有型・関数 |
| `DistanceField/GlobalDistanceFieldShared.ush` | Global SDF クリップマップサンプリング共通関数 |
| `MeshDistanceFieldCommon.ush` | Mesh SDF の符号付き距離計算共通関数 |

---

## GPU データフロー

```
【Mesh SDF アップロード（フレーム単位）】
FDistanceFieldSceneData::UpdateDistanceFieldAtlas()
  └─ ScatterUploadCS（DistanceFieldStreaming.usf）
       → Atlas Texture（Texture3D R16F/R8）へ書き込み

【Global SDF 合成（フレーム単位）】
UpdateGlobalDistanceFieldVolume()
  └─ ComposeObjectsIntoClipmap（GlobalDistanceFieldCompositeObjects.usf）
       → PageAtlasTexture（Texture3D）へ書き込み
  └─ GenerateMipCS（GlobalDistanceFieldMip.usf）
       → MipTexture（Texture3D）へ書き込み

【DFAO（フレーム単位）】
RenderDistanceFieldLighting()
  ├─ CullObjectsForViewCS（DistanceFieldObjectCulling.usf）
  ├─ ComputeDistanceFieldNormalCS（DistanceFieldScreenGridLighting.usf）
  ├─ AOLevelSetCS / ConeTraceCS（DistanceFieldScreenGridLighting.usf）
  └─ UpsampleBentNormalAO（DistanceFieldLightingPost.usf）
       → BentNormalAO テクスチャ

【DF Shadows（フレーム単位）】
RayTracedDistanceFieldProjection()
  ├─ CullObjectsToShadowFrustumCS（DistanceFieldShadowing.usf）
  └─ RayTraceShadowsCS（DistanceFieldShadowing.usf）
       → RayTracedShadowsTexture
```

---

## 各グループの詳細ドキュメント

| グループ | 詳細 | リファレンス |
|---------|------|------------|
| a: Mesh SDF | [[a_MeshSDF/detail_mesh_sdf]] | [[a_MeshSDF/ref_mesh_sdf]] |
| b: Global SDF | [[b_GlobalSDF/detail_global_sdf]] | [[b_GlobalSDF/ref_global_sdf]] |
| c: DFAO | [[c_DFAO/detail_dfao]] | [[c_DFAO/ref_dfao]] |
| d: DF Shadows | [[d_DFShadows/detail_df_shadows]] | [[d_DFShadows/ref_df_shadows]] |

---

## CPU 側対応ドキュメント

| CPU ドキュメント | 対応 GPU グループ |
|---------------|----------------|
| [[a_mesh_sdf]] — FDistanceFieldVolumeData | グループ a |
| [[b_global_sdf]] — UpdateGlobalDistanceFieldVolume | グループ b |
| [[c_dfao]] — RenderDistanceFieldLighting | グループ c |
| [[d_df_shadows]] — RayTraceShadows | グループ d |
