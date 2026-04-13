# DistanceField GPU シェーダードキュメント チェックリスト

## 概要
- [ ] `01_distance_field_gpu_overview.md` … SDF GPU シェーダー全体概要（グループ別シェーダー対応表）

---

## GPU シェーダー（グループ別）

### a: Mesh SDF ビルド・アップロード（2本）
- [ ] `a_MeshSDF/detail_mesh_sdf.md`
  - GenerateSignedDistanceFieldVolumeData（CPU生成 → GPU アップロード）
  - DistanceField Atlas Texture 形式（R16F / R8 選択）
  - Atlas パッキングシェーダー
- [ ] `a_MeshSDF/ref_mesh_sdf.md`
  - Atlas 管理関数・CVar 一覧

### b: Global SDF 生成・更新（2本）
- [ ] `b_GlobalSDF/detail_global_sdf.md`
  - ComposeObjectsIntoClipmaps CS（Mesh SDF → クリップマップボクセル化）
  - クリップマップ 4段・差分更新アルゴリズム
  - HeightField SDF の合成（地形対応）
- [ ] `b_GlobalSDF/ref_global_sdf.md`
  - `GlobalDistanceField.usf` 主要カーネル
  - `FGlobalDistanceFieldParameterData` バインド
  - CVar: `r.GlobalDistanceField.*`

### c: Distance Field AO（2本）
- [ ] `c_DFAO/detail_dfao.md`
  - Global SDF コーントレース CS（サンプリング・遮蔽積分）
  - Temporal 蓄積フィルタ
  - r.AOQuality 別パーミュテーション
- [ ] `c_DFAO/ref_dfao.md`
  - `DistanceFieldAmbientOcclusion.usf` エントリポイント
  - FAOScreenGridResources バインド
  - CVar: `r.DistanceFieldAO`, `r.AOGlobalDFStartDistance`

### d: Distance Field Shadows（2本）
- [ ] `d_DFShadows/detail_df_shadows.md`
  - Mesh SDF コーントレース CS（ライト方向ペナンブラ近似）
  - 動的 / キャッシュ済みオブジェクト SDF の使い分け
  - `r.DFShadowQuality` 別パーミュテーション
- [ ] `d_DFShadows/ref_df_shadows.md`
  - `DistanceFieldShadowing.usf` エントリポイント
  - `FDistanceFieldShadowingParameters` バインド
  - CVar: `r.DistanceFieldShadowing`, `r.DFShadowQuality`

---

合計: 概要 1 + GPU シェーダー 8 = **9 ファイル**

---

## メモ：Lumen との関係
- Lumen の SDF トレース（Surface Cache / Screen Trace）は `Lumen/` GPU フォルダで管理
- このフォルダは「Lumen 非依存の SDF 共通シェーダー」を対象とする
- Global SDF パラメータ（`FGlobalDistanceFieldParameterData`）は Lumen も DFAO も共有
