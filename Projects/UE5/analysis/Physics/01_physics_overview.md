# Physics システム全体概要

- 取得対象: `Engine/Source/Runtime/Engine/Classes/PhysicsEngine/`, `Engine/Source/Runtime/Experimental/Chaos/`
- 上位: [[UE5 解析インデックス]]

---

## UE5 Physics システムの構成

UE5 は **Chaos Physics Engine** を標準採用（UE5.0〜）。  
PhysX から完全移行し、Cloth・破壊・ビークルも Chaos ベースになった。

| 概念 | クラス | 説明 |
|------|--------|------|
| 剛体 | `FBodyInstance` | コンポーネントの物理ボディ |
| 形状 | `UBoxComponent` / `USphereComponent` 等 | コリジョン形状 |
| PhysicsAsset | `UPhysicsAsset` | スケルタルメッシュの物理設定 |
| 制約 | `FConstraintInstance` | ジョイント・ラグドール |
| 破壊 | `UGeometryCollectionComponent` | Chaos Fracture |
| クロス | `UChaosClothComponent` | 布シミュレーション |
| ビークル | `UChaosVehicleMovementComponent` | 車両物理 |
| シーン | `FChaosScene` | 物理シーン管理 |

---

## 物理更新フロー

```
UWorld::Tick()
  └─ FPhysScene_Chaos::StartFrame()
      └─ Chaos::FPBDRigidsSolver::AdvanceAndDispatch()
          ├─ CollisionDetection（BVH・GJK・EPA）
          ├─ ConstraintSolve（PBD イテレーション）
          └─ Integration（速度・位置更新）
              └─ FChaosScene::EndFrame()
                  └─ UPrimitiveComponent::SyncComponentToRBPhysics()
```

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `BodyInstance.h/.cpp` | 剛体インスタンス |
| `PhysScene_Chaos.h/.cpp` | Chaos 物理シーン |
| `PhysicsAsset.h/.cpp` | PhysicsAsset 管理 |
| `ConstraintInstance.h/.cpp` | 制約（ジョイント）|
| `GeometryCollectionComponent.h/.cpp` | 破壊メッシュ |
| `ChaosVehicleMovementComponent.h/.cpp` | ビークル移動 |

---

## サブシステムドキュメント

| ドキュメント | 内容 |
|------------|------|
| `Details/a_chaos_solver.md` | Chaos ソルバー・PBD・衝突検出 |
| `Details/b_collision.md` | コリジョン形状・チャンネル・Overlap/Hit イベント |
| `Details/c_constraints.md` | 制約・ラグドール・PhysicsAsset |
| `Details/d_destruction.md` | Chaos Fracture・GeometryCollection |
| `Details/e_vehicles.md` … | Chaos Vehicle・タイヤ・サスペンション |
