# DistanceField CPU ドキュメント チェックリスト

## 概要
- [ ] `10_distance_field_overview.md` … SDF 全体概要（利用システム対応表・データフロー・CPU/GPU役割分担）

---

## Details（4本）

### a: Mesh SDF
- [ ] `Details/a_mesh_sdf.md`
  - FDistanceFieldVolumeData 構造と Atlas へのパッキング
  - オフラインビルド（FDistanceFieldAsyncQueue / GenerateSignedDistanceFieldVolumeData）
  - ランタイムでの Mesh SDF アップロード・管理（FDistanceFieldSceneData）
  - Lumen / DFAO / DFShadows それぞれへの参照方式

### b: Global SDF
- [ ] `Details/b_global_sdf.md`
  - クリップマップ 4段構成と更新方針（カメラ追従・差分更新）
  - FGlobalDistanceFieldInfo / FGlobalDistanceFieldParameterData
  - Mesh SDF → Global SDF へのボクセル合成パス（ComposeObjects）
  - Lumen での利用（SurfaceCache Tracing）と DFAO での利用の違い

### c: Distance Field AO（DFAO）
- [ ] `Details/c_dfao.md`
  - DFAO の有効条件（r.DistanceFieldAO, Sky Light 必須）
  - GlobalSDF を使ったコーントレースによる AO 計算
  - Temporal フィルタとの統合
  - Lumen が有効な場合の DFAO 無効化ロジック

### d: Distance Field Shadows
- [ ] `Details/d_df_shadows.md`
  - Mesh SDF を使ったソフトシャドウのコーントレース
  - 動的・静的オブジェクト別の SDF キャッシュ戦略
  - `r.DistanceFieldShadowing` 有効条件と VSM/Lumen との共存

---

## Reference（4本）

### Mesh SDF / Atlas
- [ ] `Reference/ref_df_atlas.md`
  - `FDistanceFieldVolumeData` 全メンバ
  - `FDistanceFieldAsyncQueue` ビルドキュー
  - `FDistanceFieldSceneData` — MeshBuildData, ObjectBuffers, AtlasAllocation
  - `UpdateDistanceFieldAtlas()` フロー

### Global SDF
- [ ] `Reference/ref_global_sdf.md`
  - `FGlobalDistanceFieldInfo` 全メンバ
  - `UpdateGlobalDistanceField()` 関数群
  - `FGlobalDistanceFieldParameterData` シェーダーバインド
  - CVar: `r.GlobalDistanceField.*`

### DFAO エントリポイント
- [ ] `Reference/ref_dfao.md`
  - `RenderDistanceFieldLighting()` / `AddDistanceFieldAOPass()` シグネチャ
  - `FAOScreenGridResources`
  - CVar: `r.DistanceFieldAO`, `r.AOQuality`, `r.AOGlobalDFStartDistance`

### Distance Field Shadows エントリポイント
- [ ] `Reference/ref_df_shadows.md`
  - `RayTraceShadowsWithDistanceFields()` / `AddDistanceFieldShadowPass()`
  - `FDistanceFieldShadowingParameters`
  - CVar: `r.DistanceFieldShadowing`, `r.DFShadowQuality`

---

合計: 概要 1 + Details 4 + Reference 4 = **9 ファイル**

---

## メモ：関連システムの SDF 利用箇所
| システム | 利用する SDF | ソース |
|----------|------------|--------|
| Lumen | Mesh SDF + Global SDF | `Lumen/LumenSceneDirectLighting.cpp` 等 |
| DFAO | Global SDF（コーントレース） | `DistanceFieldAmbientOcclusion.cpp` |
| DF Shadows | Mesh SDF（コーントレース） | `DistanceFieldShadowing.cpp` |
| Mesh SDF ビルド | — | `DistanceFieldAtlas.cpp` |
| Global SDF 更新 | Mesh SDF を合成 | `GlobalDistanceField.cpp` |
