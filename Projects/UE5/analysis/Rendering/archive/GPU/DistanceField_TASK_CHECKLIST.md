# DistanceField GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_distance_field_gpu_overview.md` … SDF GPU シェーダー全体概要（グループ別シェーダー対応表）

---

## GPU シェーダー（グループ別）

### a: Mesh SDF ビルド・アップロード（2本）
- [x] `a_MeshSDF/detail_mesh_sdf.md`
  - ScatterUploadCS（DistanceFieldStreaming.usf）
  - DownsampleMeshSDFCS（DistanceFieldDownsampling.usf）
  - FDFObjectData / DistanceToNearestSurfaceForObject
- [x] `a_MeshSDF/ref_mesh_sdf.md`
  - Atlas テクスチャ仕様・CVar 一覧

### b: Global SDF 生成・更新（2本）
- [x] `b_GlobalSDF/detail_global_sdf.md`
  - ComposeObjectsIntoPagesCS（GlobalDistanceFieldCompositeObjects.usf）
  - ページベース Sparse 構造（PageAtlasTexture / PageTableTexture）
  - GenerateMipCS（GlobalDistanceFieldMip.usf）
  - 差分更新アルゴリズム・GDF_MostlyStatic vs GDF_Full
- [x] `b_GlobalSDF/ref_global_sdf.md`
  - GlobalDistanceField.usf 主要 CS エントリ
  - FGlobalDistanceFieldParameters2 HLSL バインド
  - GlobalDistanceFieldShared.ush 主要関数

### c: Distance Field AO（2本）
- [x] `c_DFAO/detail_dfao.md`
  - CullObjectsForViewCS（DistanceFieldObjectCulling.usf）
  - AOLevelSetCS / コーントレースループ（DistanceFieldScreenGridLighting.usf）
  - r.AOQuality パーミュテーション・Temporal 統合
- [x] `c_DFAO/ref_dfao.md`
  - FAOParameters / FDFAOUpsampleParameters / FAOScreenGridParameters バインド
  - 共有ヘッダ関数一覧・定数

### d: Distance Field Shadows（2本）
- [x] `d_DFShadows/detail_df_shadows.md`
  - CullObjectsToShadowFrustumCS（DistanceFieldShadowing.usf）
  - RayTraceShadowsCS コーントレース・ペナンブラ計算原理
  - CULLING_TYPE / DISTANCEFIELD_PRIMITIVE_TYPE パーミュテーション
- [x] `d_DFShadows/ref_df_shadows.md`
  - DistanceFieldShadowing.usf グローバルパラメータ
  - ShadowConvexHullIntersectSphere / IntersectBox
  - CVar 一覧

---

合計: 概要 1 + GPU シェーダー 8 = **9 ファイル** — 全完了 ✅

---

## メモ：Lumen との関係
- Lumen の SDF トレース（Surface Cache / Screen Trace）は `Lumen/` GPU フォルダで管理
- このフォルダは「Lumen 非依存の SDF 共通シェーダー」を対象とする
- Global SDF パラメータ（`FGlobalDistanceFieldParameterData`）は Lumen も DFAO も共有
