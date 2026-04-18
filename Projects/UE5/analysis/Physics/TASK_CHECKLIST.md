# Physics ドキュメント チェックリスト

## 概要
- [ ] `01_physics_overview.md` … Physics 全体概要（更新予定）
- [ ] `_source_map.md` … ソースマップ

---

## サブフォルダ構成

### ChaosSolver — Chaos PBD ソルバー
- [ ] `ChaosSolver/01_overview.md` … ChaosSolver 概要
- [ ] `ChaosSolver/Details/a_pbd_solver.md` … FPBDRigidsSolver・PBD イテレーション・Island 管理
- [ ] `ChaosSolver/Details/b_collision_detection.md` … BVH・GJK・EPA・Midphase/Narrowphase
- [ ] `ChaosSolver/Details/c_thread_model.md` … PhysicsThread・TaskParallelFor・島並列化
- [ ] `ChaosSolver/Details/d_solver_events.md` … FCollisionContactModifyCallback・OnBreak・OnCrumble
- [ ] `ChaosSolver/Reference/ref_solver_api.md` … FPBDRigidsSolver / FChaosScene API
- [ ] `ChaosSolver/Reference/ref_chaos_cvars.md` … p.* / Chaos.* CVar 一覧

### Collision — コリジョン
- [ ] `Collision/01_overview.md` … Collision 概要
- [ ] `Collision/Details/a_collision_shapes.md` … UBoxComponent/USphereComponent/UCapsuleComponent/UConvexMesh
- [ ] `Collision/Details/b_channels_profiles.md` … ECollisionChannel・CollisionProfile・ObjectType・ResponseContainer
- [ ] `Collision/Details/c_queries.md` … LineTrace・SweepSingle/Multi・OverlapMulti・FHitResult
- [ ] `Collision/Details/d_collision_events.md` … OnComponentHit/BeginOverlap/EndOverlap・GenerateOverlapEvents
- [ ] `Collision/Reference/ref_collision_api.md` … UPrimitiveComponent コリジョン API
- [ ] `Collision/Reference/ref_trace_api.md` … UWorld Trace/Overlap API・ECC_* 一覧

### Constraints — 制約・ラグドール
- [ ] `Constraints/01_overview.md` … Constraints 概要
- [ ] `Constraints/Details/a_constraint_instance.md` … FConstraintInstance・Linear/Angular Limits・Motor
- [ ] `Constraints/Details/b_physics_asset.md` … UPhysicsAsset・BodySetup・ConstraintProfile
- [ ] `Constraints/Details/c_ragdoll.md` … SetSimulatePhysics・BlendWeight・PhysicalAnimation
- [ ] `Constraints/Reference/ref_constraint_api.md` … FConstraintInstance / UPhysicsConstraintComponent API

### Destruction — 破壊システム
- [ ] `Destruction/01_overview.md` … Destruction 概要
- [ ] `Destruction/Details/a_geometry_collection.md` … UGeometryCollectionComponent・FGeometryCollection 構造
- [ ] `Destruction/Details/b_fracture.md` … UFractureTool・Voronoi/Planar/Cluster 分割
- [ ] `Destruction/Details/c_field_system.md` … AFieldSystemActor・RadialFalloff/UniformVector・ランタイム破壊
- [ ] `Destruction/Reference/ref_destruction_api.md` … UGeometryCollectionComponent / AFieldSystemActor API

### Vehicles — ビークル物理
- [ ] `Vehicles/01_overview.md` … Vehicles 概要
- [ ] `Vehicles/Details/a_vehicle_component.md` … UChaosVehicleMovementComponent・SimulationData
- [ ] `Vehicles/Details/b_tire_model.md` … FChaosWheelSetup・タイヤ摩擦曲線・スリップ計算
- [ ] `Vehicles/Details/c_suspension.md` … サスペンション設定・SprintRate/DampingRate・レイキャスト
- [ ] `Vehicles/Reference/ref_vehicle_api.md` … UChaosVehicleMovementComponent API

### Cloth — 布シミュレーション
- [ ] `Cloth/01_overview.md` … Cloth 概要
- [ ] `Cloth/Details/a_chaos_cloth.md` … UChaosClothComponent・ClothConfig・Simulation 設定
- [ ] `Cloth/Details/b_cloth_constraints.md` … Distance/Bend/SelfCollision 制約・Wind 影響
- [ ] `Cloth/Reference/ref_cloth_api.md` … UChaosClothComponent / UChaosClothConfig API

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| ChaosSolver | 0/1 | 0/4 | 0/2 | 0/7 |
| Collision | 0/1 | 0/4 | 0/2 | 0/7 |
| Constraints | 0/1 | 0/3 | 0/1 | 0/5 |
| Destruction | 0/1 | 0/3 | 0/1 | 0/5 |
| Vehicles | 0/1 | 0/3 | 0/1 | 0/5 |
| Cloth | 0/1 | 0/2 | 0/1 | 0/4 |

**合計**: 概要 1 + ソースマップ 1 + サブフォルダ概要 6 + Details 19 + Reference 8 = **35 ファイル**
