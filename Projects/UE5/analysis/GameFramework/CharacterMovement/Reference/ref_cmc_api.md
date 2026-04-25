# UCharacterMovementComponent API リファレンス

- 上位: [[CharacterMovement/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/CharacterMovementComponent.h`

---

## 移動モード

```cpp
// 現在モード取得・設定
TEnumAsByte<EMovementMode> MovementMode;      // UPROPERTY
uint8 CustomMovementMode;                     // UPROPERTY

virtual void SetMovementMode(EMovementMode NewMovementMode, uint8 NewCustomMode = 0);
void SetGroundMovementMode(EMovementMode NewGroundMovementMode);
EMovementMode GetGroundMovementMode() const;

virtual void OnMovementModeChanged(EMovementMode PreviousMovementMode, uint8 PreviousCustomMode);
```

---

## 主要プロパティ（速度・加速度）

```cpp
float MaxWalkSpeed           = 600.f;
float MaxWalkSpeedCrouched   = 300.f;
float MaxSwimSpeed           = 300.f;
float MaxFlySpeed            = 600.f;
float MaxAcceleration        = 2048.f;
float GroundFriction         = 8.f;
float BrakingDecelerationWalking = 2048.f;
float BrakingDecelerationFalling = 0.f;
float BrakingDecelerationFlying  = 0.f;
float BrakingDecelerationSwimming= 0.f;
float GravityScale           = 1.f;
float JumpZVelocity          = 420.f;
float AirControl             = 0.05f;
float AirControlBoostMultiplier      = 2.f;
float AirControlBoostVelocityThreshold = 25.f;
```

---

## 主要プロパティ（床・段差）

```cpp
float MaxStepHeight      = 45.f;
float WalkableFloorAngle = 44.765f;
float WalkableFloorZ;                   // WalkableFloorAngle から自動計算
float PerchRadiusThreshold = 0.f;
float PerchAdditionalHeight = 40.f;
```

---

## 物理 Phys 仮想関数

```cpp
virtual void PerformMovement(float DeltaTime);      // メイン移動処理

// モードごとの物理
virtual void PhysWalking  (float deltaTime, int32 Iterations);
virtual void PhysNavWalking(float deltaTime, int32 Iterations);
virtual void PhysFalling  (float deltaTime, int32 Iterations);
virtual void PhysSwimming (float deltaTime, int32 Iterations);
virtual void PhysFlying   (float deltaTime, int32 Iterations);
virtual void PhysCustom   (float deltaTime, int32 Iterations);  // ← カスタムで実装

// 速度計算
virtual void CalcVelocity(float DeltaTime, float Friction, bool bFluid, float BrakingDeceleration);
```

---

## 床検出

```cpp
FFindFloorResult CurrentFloor;   // 現在の床情報（毎フレーム更新）

virtual void FindFloor(const FVector& CapsuleLocation,
                        FFindFloorResult& OutFloorResult,
                        bool bCanUseCachedLocation,
                        const FHitResult* DownwardSweepResult = nullptr) const;

// 歩行可能判定
virtual bool IsWalkable(const FHitResult& Hit) const;

// 着地判定
virtual bool IsValidLandingSpot(const FVector& CapsuleLocation,
                                  const FHitResult& Hit) const;
```

---

## 段差・スライド

```cpp
virtual bool CanStepUp(const FHitResult& Hit) const;
virtual bool StepUp(const FVector& GravDir, const FVector& Delta,
                     const FHitResult& Hit,
                     FStepDownResult* OutStepDownResult = nullptr);

virtual float SlideAlongSurface(const FVector& Delta, float Time,
                                  const FVector& Normal,
                                  FHitResult& Hit, bool bHandleImpact) override;

virtual void MoveAlongFloor(const FVector& InVelocity, float DeltaSeconds,
                              FStepDownResult* OutStepDownResult = nullptr);
```

---

## RootMotion

```cpp
FRootMotionSourceGroup CurrentRootMotion;  // 実行中の RootMotionSource 群
FRootMotionMovementParams RootMotionParams; // AnimRootMotion の変位

bool HasRootMotionSources() const;
bool HasAnimRootMotion() const;  // AnimMontage の RootMotion 実行中か

uint16 ApplyRootMotionSource(TSharedPtr<FRootMotionSource> SourcePtr);
void RemoveRootMotionSource(FName InstanceName);
void RemoveRootMotionSourceByID(uint16 RootMotionSourceID);

virtual void ApplyRootMotionToVelocity(float deltaTime);

// AnimMontage のスケール
void  SetRootMotionTranslationScaleFactor(float InScale);
float GetRootMotionTranslationScaleFactor() const;
```

---

## ネット予測

```cpp
// クライアント予測データ取得
virtual FNetworkPredictionData_Client* GetPredictionData_Client() const override;
FNetworkPredictionData_Client_Character* GetPredictionData_Client_Character() const;

// 位置補間（SimulatedProxy）
virtual void SmoothClientPosition(float DeltaSeconds);
```

---

## ジャンプ・Launch

```cpp
// ACharacter 経由で呼ぶ
void LaunchCharacter(FVector LaunchVelocity, bool bXYOverride, bool bZOverride);

// CMC 直接 API
virtual void Launch(FVector const& LaunchVel);
virtual bool HandlePendingLaunch();
FVector PendingLaunchVelocity;  // LaunchCharacter で設定され次フレームに適用

float GetMaxJumpHeight() const;
float GetMaxJumpHeightWithJumpTime() const;
```

---

## 物理インタラクション

```cpp
uint8 bEnablePhysicsInteraction:1;   // 物理オブジェクトに力を加えるか

float InitialPushForceFactor  = 500.f;
float PushForceFactor         = 750000.f;
float RepulsionForce          = 2.5f;
float StandingDownwardForceScale = 1.f;
bool  bScalePushForceToVelocity = true;

virtual void ApplyRepulsionForce(float DeltaSeconds);

void AddImpulse(FVector Impulse, bool bVelocityChange=false);
void AddForce(FVector Force);
void ClearAccumulatedForces();
```

---

## スムージングモード

```cpp
UPROPERTY(EditDefaultsOnly)
ENetworkSmoothingMode NetworkSmoothingMode;
// Linear / Exponential / Replay / Disabled
```

---

## 関連ドキュメント

- [[Details/a_movement_modes]] — 各モードの物理処理
- [[Details/b_cmc_networking]] — SavedMove・ServerMove 詳細
- [[Details/c_root_motion]] — RootMotionSource 詳細
- [[Details/d_physics_interaction]] — StepUp・床チェック詳細
- [[ref_movement_flags]] — EMovementMode 一覧
