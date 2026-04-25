# 物理インタラクション — StepUp / Floor チェック / Slide / Launch

- 上位: [[CharacterMovement/01_overview]]
- 関連: [[a_movement_modes]] | [[b_cmc_networking]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/CharacterMovementComponent.h`, `Components/CharacterMovementComponent.cpp`

---

## 概要

CMC はカプセルコリジョンを使った離散的な物理処理を毎フレーム行う。床検出（FindFloor）・段差登り（StepUp）・スライド（SlideAlongSurface）・物理オブジェクトへの力付与（PhysicsInteraction）・打ち上げ（Launch）が主な機能。

---

## FFindFloorResult — 床検出結果

```cpp
struct FFindFloorResult
{
    bool  bBlockingHit      = false;  // 地面ヒットあり
    bool  bWalkableFloor    = false;  // 歩行可能な床か
    bool  bLineTrace        = false;  // ラインのみでヒット（スイープ失敗時）
    float FloorDist         = 0.f;   // カプセル底面から床までの距離
    float LineDist          = 0.f;   // ラインヒットの距離
    FHitResult HitResult;            // 衝突情報
};
```

---

## FindFloor のフロー

```
PhysWalking() → MoveAlongFloor() 後
  └─ FindFloor(CapsuleLocation, OutFloor, bCanUseCachedLocation)
       ├─ ComputeFloorDist()
       │    ├─ CapsuleSweep（垂直下向き）
       │    │    └─ ヒット → FFindFloorResult.bBlockingHit = true
       │    └─ スイープ失敗時は LineTrace でフォールバック
       │         └─ bLineTrace = true（着地判定には使えない）
       └─ IsWalkable(HitResult)
            └─ HitResult.Normal.Z >= WalkableFloorZ → bWalkableFloor = true
```

### IsWalkable の条件

```cpp
// 斜面角度チェック
// WalkableFloorAngle = 44.765° → WalkableFloorZ ≒ 0.71
bool IsWalkable = HitNormal.Z >= WalkableFloorZ;
// 急斜面・垂直面・天井はすべて false
```

---

## StepUp — 段差登り

```
PhysWalking() → SafeMoveUpdatedComponent() で壁にヒット
  └─ CanStepUp(Hit) == true?
       ├─ 障害物の高さ <= MaxStepHeight (45cm) → StepUp 試行
       └─ StepUp(GravDir, Delta, Hit)          [CMC.cpp]
            ├─ 1) キャラを MaxStepHeight 分上に移動
            ├─ 2) 元の移動方向へ SafeMoveUpdatedComponent
            ├─ 3) 成功なら FindFloor で新しい床を確認
            │    └─ IsWalkable → StepUp 確定
            └─ 4) 失敗 or 床がない → 元の位置に RevertMove
```

```cpp
// 段差の設定
CMC->MaxStepHeight = 45.f;   // この高さ以下の段差を自動で登る

// CanStepUp をオーバーライドして特定 Actor の上に乗れないようにする
virtual bool CanStepUp(const FHitResult& Hit) const override
{
    if (Hit.GetActor() && Hit.GetActor()->ActorHasTag(TEXT("NoStep")))
        return false;
    return Super::CanStepUp(Hit);
}
```

---

## SlideAlongSurface — 壁スライド

```cpp
// 壁に当たった時に沿って動く（標準の UMovementComponent の実装を上書き）
virtual float SlideAlongSurface(const FVector& Delta, float Time,
                                 const FVector& Normal,
                                 FHitResult& Hit, bool bHandleImpact) override;
```

Walking 時は**壁に沿って床方向に引き寄せ**（斜面への潜り込み防止）し、Falling 時は**純粋に壁面方向に反射**する動作が変わる。

---

## PhysicsInteraction — 物理オブジェクトへの力付与

キャラクターが物理シミュレーション中のオブジェクト（家具・箱等）に当たったとき自動で力を加える:

```cpp
// 有効化
CMC->bEnablePhysicsInteraction = true;

// 押す力の設定
CMC->InitialPushForceFactor  = 500.f;   // 最初に加える力の倍率
CMC->PushForceFactor         = 750000.f;// 継続的な押す力
CMC->RepulsionForce          = 2.5f;    // キャラクターが上に立つ時の押し下げ力
CMC->StandingDownwardForceScale = 1.f;  // 上に立つ時に加える重量倍率

// 速度に応じて押す力をスケール
CMC->bScalePushForceToVelocity = true;

// 質量に応じてスケール
CMC->bPushForceScaledToMass = false;   // 質量無視でフラットに加える
```

---

## LaunchCharacter — 打ち上げ・吹き飛び

```cpp
// ACharacter から直接呼ぶ
Character->LaunchCharacter(
    FVector(0.f, 0.f, 1000.f),  // 打ち上げ速度
    bXYOverride=false,           // true なら水平速度を LaunchVelocity に置き換え
    bZOverride=true              // true なら垂直速度を置き換え（false なら加算）
);

// 内部フロー
// ACharacter::LaunchCharacter()
//   → CMC->Launch(LaunchVel)
//        → PendingLaunchVelocity = LaunchVel
//   → PerformMovement() → HandlePendingLaunch()
//        → Velocity += PendingLaunchVelocity
//        → SetMovementMode(MOVE_Falling)
```

---

## AddImpulse / AddForce

```cpp
// 瞬間的な力（次フレームに一度だけ適用）
CMC->AddImpulse(FVector(500.f, 0.f, 0.f), bVelocityChange=true);

// 継続的な力（TickComponent のたびに加算）
CMC->AddForce(FVector(0.f, 0.f, -500.f));  // 追加重力など

// 蓄積された力は ApplyAccumulatedForces() で Velocity に適用後クリア
```

---

## パーチ (Perch) — 縁のギリギリ立ち

```cpp
// 縁で立ち続けられる許容半径
CMC->PerchRadiusThreshold = 0.f;     // 0 = Perch なし（デフォルト）
                                     // > 0 = カプセル縁から X cm 以内なら落ちない

CMC->PerchAdditionalHeight = 40.f;  // パーチ中に有効な MaxStepHeight の追加分
```

---

## 落下チェック — ShouldCatchAir / HandleWalkingOffLedge

```cpp
// 崖の端から飛び出した瞬間を検出（慣性ジャンプに使うことも）
virtual bool ShouldCatchAir(const FFindFloorResult& OldFloor,
                             const FFindFloorResult& NewFloor) override
{
    // デフォルト: false（落ちる）
    // true を返すと MOVE_Falling へ遷移せずその場で止まる
    return Super::ShouldCatchAir(OldFloor, NewFloor);
}

// 崖から歩き出した時のコールバック
virtual void HandleWalkingOffLedge(
    const FVector& PreviousFloorImpactNormal,
    const FVector& PreviousFloorContactNormal,
    const FVector& PreviousLocation,
    float TimeDelta) override
{
    // ここで追加水平速度などを加えられる（慣性付きの崖落下）
}
```

---

## 関連 CVar

| CVar | 説明 |
|------|------|
| `p.StepUpDistance 1` | StepUp 時のデバッグ描画 |
| `showdebug CharacterMovement` | CMC 全体のデバッグ表示 |
| `p.VisualizeMovement 1` | 移動ベクトル可視化 |
| `p.netshowcorrections 1` | ネット補正位置の可視化 |

---

## 関連ドキュメント

- [[a_movement_modes]] — PhysWalking / PhysFalling のフロー
- [[b_cmc_networking]] — ネット同期での床検出・補正
- [[Reference/ref_cmc_api]] — UCharacterMovementComponent API
- [[Reference/ref_movement_flags]] — EMovementMode / 移動フラグ一覧
