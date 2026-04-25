# 移動モード — Walking / Falling / Swimming / Flying / Custom

- 上位: [[CharacterMovement/01_overview]]
- 関連: [[b_cmc_networking]] | [[c_root_motion]] | [[d_physics_interaction]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/CharacterMovementComponent.h`, `CharacterMovementComponent.cpp`

---

## 概要

`UCharacterMovementComponent`（CMC）は `EMovementMode` に基づいて毎フレーム物理計算を切り替える。モードごとに専用の `Phys*()` 関数が呼ばれ、速度計算・衝突応答・床検出が独立して動作する。

---

## EMovementMode 一覧

```cpp
UENUM(BlueprintType)
enum EMovementMode : int
{
    MOVE_None        = 0,  // 移動不可（ラグドール等）
    MOVE_Walking     = 1,  // 地面歩行（床検出あり）
    MOVE_NavWalking  = 2,  // NavMesh ベース歩行（精度は低いが軽量）
    MOVE_Falling     = 3,  // 落下・ジャンプ中
    MOVE_Swimming    = 4,  // 水中（PhysicsVolume）
    MOVE_Flying      = 5,  // 飛行（重力無効）
    MOVE_Custom      = 6,  // カスタム移動（CustomMovementMode で番号管理）
};
```

カスタムモード番号は `CustomMovementMode` (uint8) で指定:

```cpp
// カスタム移動モードへ切り替え
MovementComponent->SetMovementMode(MOVE_Custom, /*CustomMode=*/ 1);

// 判定
if (MovementComponent->MovementMode == MOVE_Custom &&
    MovementComponent->CustomMovementMode == 1)
{
    // カスタムモード 1 の処理
}
```

---

## 主要プロパティ

```cpp
// ─── 速度上限 ──────────────────────────────────────────
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Character)
float MaxWalkSpeed          = 600.f;    // 歩き最大速度
float MaxWalkSpeedCrouched  = 300.f;    // しゃがみ最大速度
float MaxSwimSpeed          = 300.f;    // 水中最大速度
float MaxFlySpeed           = 600.f;    // 飛行最大速度
float MaxAcceleration       = 2048.f;   // 最大加速度

// ─── 摩擦・制動 ─────────────────────────────────────────
float GroundFriction            = 8.f;
float BrakingDecelerationWalking= 2048.f;
float BrakingDecelerationFalling= 0.f;   // 空中は制動なし（デフォルト）

// ─── 重力・ジャンプ ─────────────────────────────────────
float GravityScale   = 1.f;             // 重力倍率
float JumpZVelocity  = 420.f;           // ジャンプ初速
float AirControl     = 0.05f;           // 空中での水平制御割合

// ─── 床判定 ─────────────────────────────────────────────
float MaxStepHeight      = 45.f;        // 登れる段差の高さ
float WalkableFloorAngle = 44.765f;     // 歩ける斜面の最大角度
float WalkableFloorZ;                   // WalkableFloorAngle から計算される Z 法線閾値
```

---

## 各モードの Phys 関数

```
TickComponent() → PerformMovement()
  └─ switch(MovementMode):
       MOVE_Walking    → PhysWalking(dt, iter)
       MOVE_NavWalking → PhysNavWalking(dt, iter)
       MOVE_Falling    → PhysFalling(dt, iter)
       MOVE_Swimming   → PhysSwimming(dt, iter)
       MOVE_Flying     → PhysFlying(dt, iter)
       MOVE_Custom     → PhysCustom(dt, iter)   ← virtual でオーバーライド可
```

---

## PhysWalking の処理概要

```
PhysWalking(deltaTime, Iterations)       [CharacterMovementComponent.cpp]
  ├─ CalcVelocity()                      — 加速・摩擦・制動で速度更新
  ├─ MoveAlongFloor()                    — 床面に沿って移動
  │    ├─ SafeMoveUpdatedComponent()     — カプセル移動（衝突あれば HitResult）
  │    ├─ SlideAlongSurface()            — 壁/障害物に沿ってスライド
  │    └─ StepUp()                       — 段差を登る
  ├─ FindFloor()                         — 新しい床を検出
  │    └─ FloorSweepTest()               — カプセルスイープ
  └─ AdjustFloorHeight()                 — 床との距離を微調整（浮き防止）
```

---

## PhysFalling — 空中移動

```
PhysFalling(deltaTime, Iterations)
  ├─ CalcVelocity() に BrakingDecelerationFalling を適用
  ├─ LimitAirControl()                   — 空中の水平入力を AirControl で制限
  ├─ SafeMoveUpdatedComponent()
  │    └─ 衝突 → ProcessLanded()         — 着地時に SetMovementMode(MOVE_Walking)
  └─ CheckFall()                         — 落下継続チェック
```

---

## SetMovementMode の通知フロー

```cpp
// モード変更
CMC->SetMovementMode(MOVE_Falling);

// → OnMovementModeChanged() 呼び出し（virtual）
// → ACharacter::OnMovementModeChanged() が Blueprint イベントを発火
// → K2_OnMovementModeChanged (Blueprint) に通知
```

---

## カスタム移動モードの実装例

```cpp
// MyCharacterMovementComponent.h
UCLASS()
class UMyCharacterMovementComponent : public UCharacterMovementComponent
{
    GENERATED_BODY()

    // カスタムモード番号
    enum ECustomMovementMode : uint8
    {
        CMOVE_WallRun = 0,
        CMOVE_Dash    = 1,
    };

protected:
    virtual void PhysCustom(float deltaTime, int32 Iterations) override;
    virtual void OnMovementModeChanged(EMovementMode PreviousMovementMode,
                                       uint8 PreviousCustomMode) override;
};

// MyCharacterMovementComponent.cpp
void UMyCharacterMovementComponent::PhysCustom(float deltaTime, int32 Iterations)
{
    Super::PhysCustom(deltaTime, Iterations);

    switch (CustomMovementMode)
    {
    case CMOVE_WallRun:
        PhysWallRun(deltaTime, Iterations);
        break;
    case CMOVE_Dash:
        PhysDash(deltaTime, Iterations);
        break;
    }
}

void UMyCharacterMovementComponent::OnMovementModeChanged(EMovementMode Prev, uint8 PrevCustom)
{
    Super::OnMovementModeChanged(Prev, PrevCustom);

    if (Prev == MOVE_Custom && PrevCustom == CMOVE_WallRun)
    {
        // ウォールランから抜けた時の後処理
    }
}
```

---

## NavWalking vs Walking

| 項目 | MOVE_Walking | MOVE_NavWalking |
|------|-------------|-----------------|
| 床検出 | カプセルスイープ | NavMesh 投影 |
| 精度 | 高い（物理ベース） | 低い（メッシュ近似） |
| 用途 | プレイヤー・重要な AI | 大量の背景 AI |
| コスト | 高い | 低い |

---

## 関連ドキュメント

- [[b_cmc_networking]] — クライアント予測・ServerMove
- [[c_root_motion]] — AnimMontage による移動
- [[d_physics_interaction]] — StepUp・床チェック詳細
- [[Reference/ref_cmc_api]] — UCharacterMovementComponent API
