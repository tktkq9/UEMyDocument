# Depth Pre-pass / Velocity Buffer GPU シェーダードキュメント チェックリスト

## 概要
- [x] `01_depthprepass_gpu_overview.md` … Depth Pre-pass + Velocity GPU 全処理実行順 + CPU対応表

---

## Depth Pre-pass GPU シェーダー（グループ別）

### a: DepthOnly（2本）
- [x] `a_DepthOnly/detail_depth_only.md`  … PositionOnly VS による深度書き込みパス（Opaque / Masked）
- [x] `a_DepthOnly/ref_depth_only.md`     … DepthOnlyVertexShader#Main / DepthOnlyPixelShader#Main エントリポイント

### b: Velocity（2本）
- [x] `b_Velocity/detail_velocity.md`  … Velocity Buffer 生成（スキン / 静的 / Nanite 各メッシュ）
- [x] `b_Velocity/ref_velocity.md`     … VelocityShader MainVertexShader / MainPixelShader エントリポイント

---

合計: 概要 1 + Depth Pre-pass シェーダー 4 = **5 ファイル** ✅
