# Physics ドキュメント チェックリスト

## 概要
- [ ] `01_physics_overview.md` … Physics システム全体概要

---

## Details（詳細解析）

- [ ] `Details/a_chaos_solver.md` … Chaos PBD ソルバー・衝突検出（GJK/EPA/BVH）・スレッドモデル
- [ ] `Details/b_collision.md` … コリジョン形状・チャンネル・プロファイル・Overlap/Hit イベント
- [ ] `Details/c_constraints.md` … FConstraintInstance・ラグドール・PhysicsAsset 設定
- [ ] `Details/d_destruction.md` … GeometryCollection・Fracture・フィールドシステム
- [ ] `Details/e_vehicles.md` … Chaos Vehicle・タイヤモデル・サスペンション・ホイールコントローラー

## Reference（リファレンス）

- [ ] `Reference/ref_physics_api.md` … FBodyInstance / UPrimitiveComponent 物理 API
- [ ] `Reference/ref_collision_api.md` … LineTrace/SweepTrace/OverlapTest API・ECC_*
- [ ] `Reference/ref_chaos_cvars.md` … p.* / Chaos.* CVar 一覧

---

合計: 概要 1 + Details 5 + Reference 3 = **9 ファイル**
