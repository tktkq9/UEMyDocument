# AController / APlayerController / AAIController / APawn API リファレンス

- 上位: [[Controller/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/Controller.h`, `PlayerController.h`, `AIModule/Classes/AIController.h`, `GameFramework/Pawn.h`

---

## AController — 基底クラス

```cpp
// ─── ポゼッション ───────────────────────────────────────────
// NOTE: Possess / UnPossess は final。オーバーライドは OnPossess / OnUnPossess を使う
void Possess(APawn* InPawn);       // サーバーのみ
void UnPossess();

virtual void OnPossess(APawn* InPawn);   // ← カスタムはここで
virtual void OnUnPossess();

APawn* GetPawn() const;
APawn* K2_GetPawn() const;  // Blueprint 用（null チェック付き）

// ─── 状態 ────────────────────────────────────────────────
void     ChangeState(FName NewState);
FName    GetStateName() const;
bool     IsInState(FName StateName) const;

// ─── 視点 ────────────────────────────────────────────────
virtual AActor* GetViewTarget() const;
FRotator GetControlRotation() const;
void     SetControlRotation(const FRotator& NewRotation);

// ─── PlayerState ──────────────────────────────────────────
UPROPERTY(BlueprintReadOnly)
TObjectPtr<APlayerState> PlayerState;

// ─── ネット ──────────────────────────────────────────────
bool IsLocalController() const;
bool IsLocalPlayerController() const;  // PlayerController のみ true
bool IsPlayerController() const;
```

---

## APlayerController — 入力

```cpp
// 入力モード
void SetInputMode(const FInputModeDataBase& InData);
// 具体値: FInputModeGameOnly / FInputModeUIOnly / FInputModeGameAndUI

// InputComponent スタック
void PushInputComponent(UInputComponent* Input);
bool PopInputComponent(UInputComponent* Input);
bool IsInputComponentInStack(const UInputComponent* Input) const;

// 有効化・無効化
void EnableInput(APlayerController* PlayerController);
void DisableInput(APlayerController* PlayerController);

// 入力値
virtual void AddPitchInput(float Val);
virtual void AddYawInput(float Val);
virtual void AddRollInput(float Val);
```

---

## APlayerController — カメラ

```cpp
UPROPERTY(BlueprintReadOnly)
TObjectPtr<APlayerCameraManager> PlayerCameraManager;

// ViewTarget 変更
virtual void SetViewTargetWithBlend(AActor* NewViewTarget,
                                     float BlendTime = 0,
                                     EViewTargetBlendFunction BlendFunc = VTBlend_Linear,
                                     float BlendExp = 0,
                                     bool bLockOutgoing = false);
AActor* GetViewTarget() const;

// カメラ管理
UPROPERTY(EditAnywhere)
bool bAutoManageActiveCameraTarget;  // Possess 時に自動 ViewTarget 設定

void AutoManageActiveCameraTarget(AActor* SuggestedTarget);
```

---

## APlayerController — HUD / UI

```cpp
UPROPERTY()
TObjectPtr<AHUD> MyHUD;

// サーバーからクライアントへ HUD クラスを送信
void ClientSetHUD(TSubclassOf<AHUD> NewHUDClass);

// ローカル判定（HUD 更新は IsLocalPlayerController() が true の時のみ）
bool IsLocalPlayerController() const;
```

---

## APlayerController — ネット / ポゼッション確認

```cpp
UPROPERTY()
TObjectPtr<APawn> AcknowledgedPawn;  // クライアントが確認済みの Pawn

void ServerAcknowledgePossession(APawn* P);  // クライアント → サーバー RPC

virtual void BeginPlayingState();    // NAME_Playing 状態開始時

// SpectatorPawn
ASpectatorPawn* GetSpectatorPawn() const;
UPROPERTY(EditAnywhere)
TSubclassOf<ASpectatorPawn> SpectatorClass;
```

---

## AAIController — AI ロジック

```cpp
// BT / Blackboard
virtual bool RunBehaviorTree(UBehaviorTree* BTAsset);
bool UseBlackboard(UBlackboardData* BlackboardAsset,
                   UBlackboardComponent*& BlackboardComponent);
virtual bool InitializeBlackboard(UBlackboardComponent& BlackboardComp,
                                   UBlackboardData& BlackboardAsset);

UBlackboardComponent* GetBlackboardComponent();
UBrainComponent*      GetBrainComponent() const;

// 移動
EPathFollowingRequestResult::Type MoveToActor(AActor* Goal,
    float AcceptanceRadius = -1,
    bool bStopOnOverlap = true,
    bool bUsePathfinding = true,
    bool bCanStrafe = true,
    TSubclassOf<UNavigationQueryFilter> FilterClass = {},
    bool bAllowPartialPath = true);

EPathFollowingRequestResult::Type MoveToLocation(const FVector& Dest,
    float AcceptanceRadius = -1, ...);

virtual void StopMovement() override;
EPathFollowingStatus::Type GetMoveStatus() const;

// フォーカス
void SetFocus(AActor* NewFocus, EAIFocusPriority::Type Priority = EAIFocusPriority::Gameplay);
void SetFocalPoint(FVector NewFP, EAIFocusPriority::Type Priority = EAIFocusPriority::Gameplay);
void ClearFocus(EAIFocusPriority::Type Priority);
FVector GetFocalPoint() const;
```

---

## APawn — Controller 連携

```cpp
// Controller
AController* GetController() const;
void PossessedBy(AController* NewController);   // Controller から呼ばれる
void UnPossessed();

// 入力
virtual void SetupPlayerInputComponent(UInputComponent* PlayerInputComponent);
void EnableInput(APlayerController* PlayerController);
void DisableInput(APlayerController* PlayerController);

// 移動
virtual UPawnMovementComponent* GetMovementComponent() const;
virtual FVector GetVelocity() const;

// ローカル判定
bool IsLocallyControlled() const;
bool IsPlayerControlled() const;
bool IsBot() const;  // AIController を持つか

// PlayerState
UPROPERTY(BlueprintReadOnly)
TObjectPtr<APlayerState> PlayerState;
```

---

## 関連ドキュメント

- [[Details/a_player_controller]] — PlayerController の入力・カメラ詳細
- [[Details/b_ai_controller]] — AIController の BT / 知覚
- [[Details/c_possession]] — Possess / UnPossess フロー
