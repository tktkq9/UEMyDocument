# MeshPassProcessor TASK CHECKLIST

## Phase 1/2 — Details & Reference 作成

### Details
- [x] a_mpp_pipeline    : MeshPass パイプライン全体・FMeshPassProcessor 基底
- [x] b_mpp_drawcommand : FMeshDrawCommand 構造・ソートキー
- [x] c_mpp_shaders     : FMeshMaterialShader・バインディング
- [x] d_mpp_passes      : EMeshPass enum・各パス詳細

### Reference
- [x] ref_mpp_processor  : FMeshPassProcessor API
- [x] ref_mpp_drawcommand: FMeshDrawCommand API
- [x] ref_mpp_shaders    : シェーダー関連型
- [x] ref_mpp_utils      : SimpleMeshDrawCommandPass・ユーティリティ

## Phase 3 — コード実行フロー追加・Reference 強化

### Details（`## コード実行フロー` 追加）
- [x] 11_mpp_overview    : 静的/動的コマンドの全体フロー + 関与クラス表
- [x] a_mpp_pipeline     : AddMeshBatch → BuildMeshDrawCommands → FinalizeCommand フロー
- [x] b_mpp_drawcommand  : 静的キャッシュ生成 / SubmitDraw フロー
- [x] c_mpp_shaders      : GetShaderBindings チェーン フロー
- [x] d_mpp_passes       : AddSimpleMeshPass / 静的コマンド利用パス フロー

### Reference（メンバ変数テーブル化・`> [!note]-` 追加）
- [x] ref_mpp_processor  : メンバ変数テーブル化・EMeshPassFeatures テーブル・3 callout 追加
- [x] ref_mpp_drawcommand: 3 callout 追加（生成フロー・動的インスタンシング・キャッシュライフサイクル）
- [x] ref_mpp_shaders    : EShaderFrequency テーブル化・ShaderElementData テーブル・3 callout 追加
- [x] ref_mpp_utils      : FMeshBatch メンバ変数テーブル・3 callout 追加
