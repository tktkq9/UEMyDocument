# AAIController — AI コントローラー

- 上位: [[Controller/01_overview]]
- 関連: [[a_player_controller]] | [[c_possession]]
- ソース: `Engine/Source/Runtime/AIModule/Classes/AIController.h`, `Engine/Source/Runtime/AIModule/Private/AIController.cpp`

---

## 概要

`AAIController` は AI キャラクターを操作する Controller。`BrainComponent`（BehaviorTree / StateTree）から意思決定を受け取り、`PathFollowingComponent` でナビメッシュを使った移動を実行する。`APlayerController` と同じ `AController` 継承だが、入力処理は持たず AI ロジック系のコンポーネントを保有する。

---

## クラス構造

```cpp
class AAIController : public AController,
                      public IAIPerceptionListenerInterface,
                      public IGameplayTaskOwnerInterface,
                      public IGenericTeamAgentInterface
{
protected:
    // ─── 意思決定 ─────────────────────────────────────────────
    UPROPERTY(BlueprintReadWrite, Category = AI)
    TObjectPtr<UBrainComponent>         BrainComponent;       // BT / StateTree 実行エンジン

    UPROPERTY(BlueprintReadOnly, Category = AI)
    TObjectPtr<UBlackboardComponent>    Blackboard;           // BT 共有データ

    // ─── 移動 ────────────────────────────────────────────────
    UPROPERTY(VisibleDefaultsOnly, Category = AI)
    TObjectPtr<UPathFollowingComponent> PathFollowingComponent;

    // ─── 知覚 ────────────────────────────────────────────────
    UPROPERTY(VisibleDefaultsOnly, Category = AI)
    TObjectPtr<UAIPerceptionComponent>  PerceptionComponent;

    // ─── フラグ ──────────────────────────────────────────────
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = AI)
    uint32 bStartAILogicOnPossess : 1;   // Possess 時に AI ロジック開始
    uint32 bStopAILogicOnUnposses : 1;   // UnPossess 時に AI ロジック停止
    uint32 bWantsPlayerState      : 1;   // AI に PlayerState を付与するか
};
```

---

## RunBehaviorTree / UseBlackboard

```cpp
// BT アセットを実行（内部で UseBlackboard も呼ぶ）
bool AAIController::RunBehaviorTree(UBehaviorTree* BTAsset)
//  → UBehaviorTreeComponent::StartTree() を呼び BT ルートから実行開始

// Blackboard を手動でセットアップしてから BT を走らせるパターン
UBlackboardComponent* BBComp = nullptr;
if (UseBlackboard(BBData, BBComp))
{
    RunBehaviorTree(BTAsset);
}
```

Blueprint から:
```
BeginPlay
  └─ [AIController] Run Behavior Tree (BTAsset)
```

---

## Possess 時のフロー

```
AController::Possess(InPawn)              [Controller.cpp]
  └─ AAIController::OnPossess(InPawn)     [AIController.cpp]
       ├─ Super::OnPossess(InPawn)         — Pawn::PossessedBy / SetControlRotation
       ├─ if (bStartAILogicOnPossess)
       │    └─ BrainComponent->StartLogic()
       └─ PerceptionComponent->RequestStimuliListenerUpdate()
```

---

## 移動命令

```cpp
// Actor への移動（ナビメッシュ使用）
EPathFollowingRequestResult::Type Result =
    AIController->MoveToActor(TargetActor, AcceptanceRadius=100.f);

// 座標への移動
AIController->MoveToLocation(TargetLocation, AcceptanceRadius=50.f);

// 移動停止
AIController->StopMovement();

// 移動状態を確認
EPathFollowingStatus::Type Status = AIController->GetMoveStatus();
// Moving / Idle / Waiting / Paused
```

戻り値 `EPathFollowingRequestResult`:

| 値 | 意味 |
|----|------|
| `AlreadyAtGoal` | すでに目標地点にいる |
| `RequestSuccessful` | パス探索を要求した |
| `Failed` | ナビメッシュが見つからない等でリクエスト失敗 |

---

## フォーカス制御

AI キャラクターの向きを制御する仕組み:

```cpp
// 優先度付きフォーカス設定（高い優先度が勝つ）
namespace EAIFocusPriority
{
    enum Type { Default = 0, Move = 1, Gameplay = 2 };
}

// フォーカスを Actor に設定
AIController->SetFocus(EnemyActor, EAIFocusPriority::Gameplay);

// 座標にフォーカス
AIController->SetFocalPoint(TargetLocation);

// クリア
AIController->ClearFocus(EAIFocusPriority::Gameplay);

// 取得
FVector FP = AIController->GetFocalPoint();
```

---

## AI Perception の活用

```cpp
// AMyAIController.h
UCLASS()
class AMyAIController : public AAIController
{
    GENERATED_BODY()
public:
    AMyAIController();

protected:
    virtual void OnPossess(APawn* InPawn) override;

    UFUNCTION()
    void OnTargetPerceived(AActor* Actor, FAIStimulus Stimulus);
};

// AMyAIController.cpp
AMyAIController::AMyAIController()
{
    // Perception コンポーネント設定
    PerceptionComponent = CreateDefaultSubobject<UAIPerceptionComponent>(TEXT("PerceptionComp"));

    UAISenseConfig_Sight* SightConfig = CreateDefaultSubobject<UAISenseConfig_Sight>(TEXT("SightConfig"));
    SightConfig->SightRadius = 1000.f;
    SightConfig->PeripheralVisionAngleDegrees = 60.f;
    PerceptionComponent->ConfigureSense(*SightConfig);
    PerceptionComponent->SetDominantSense(SightConfig->GetSenseImplementation());
}

void AMyAIController::OnPossess(APawn* InPawn)
{
    Super::OnPossess(InPawn);
    PerceptionComponent->OnTargetPerceptionUpdated.AddDynamic(this, &AMyAIController::OnTargetPerceived);
}
```

---

## Blackboard キーの読み書き

```cpp
// Blackboard から値を取得
UBlackboardComponent* BB = GetBlackboardComponent();

AActor* Target = Cast<AActor>(BB->GetValueAsObject(TEXT("TargetActor")));
FVector PatrolPoint = BB->GetValueAsVector(TEXT("PatrolPoint"));
bool bAlerted = BB->GetValueAsBool(TEXT("IsAlerted"));

// 書き込み
BB->SetValueAsObject(TEXT("TargetActor"), EnemyActor);
BB->SetValueAsBool(TEXT("IsAlerted"), true);
```

---

## カスタム AIController の実装例

```cpp
UCLASS()
class AMyAIController : public AAIController
{
    GENERATED_BODY()
public:
    AMyAIController();
    virtual void OnPossess(APawn* InPawn) override;

protected:
    UPROPERTY(EditDefaultsOnly, Category = AI)
    TObjectPtr<UBehaviorTree> DefaultBehaviorTree;
};

void AMyAIController::OnPossess(APawn* InPawn)
{
    Super::OnPossess(InPawn);

    if (DefaultBehaviorTree)
    {
        RunBehaviorTree(DefaultBehaviorTree);
    }
}
```

---

## 関連 CVar / デバッグ

```
showdebug AI            — AIController の状態表示
showdebug BehaviorTree  — BT ノードのデバッグ
showdebug Navigation    — パス表示
ai.debug.BrainLogLevel 1
```

---

## コード実行フロー

### Possess → RunBehaviorTree

```
AController::Possess(InPawn)                         [Controller.cpp]
  └─ AAIController::OnPossess(InPawn)               [AIController.cpp]
       ├─ Super::OnPossess()                         ← Pawn::PossessedBy / SetPawn
       ├─ if (bStartAILogicOnPossess)
       │    └─ BrainComponent->StartLogic()          ← BT 実行開始
       └─ PerceptionComponent->RequestStimuliListenerUpdate()

AAIController::RunBehaviorTree(BTAsset)             [AIController.cpp]
  └─ UseBlackboard(BTAsset->BlackboardAsset, BBComp)
       └─ InitializeBlackboard(BBComp, BBAsset)
  └─ UBehaviorTreeComponent::StartTree(BTAsset)     ← ルートノードから実行
```

### MoveToActor フロー

```
AAIController::MoveToActor(Goal, Radius, ...)        [AIController.cpp]
  └─ BuildPathfindingQuery()
  └─ PathFollowingComponent->RequestMove(MoveRequest, Path)
       └─ NavigationSystem::FindPathSync()           ← NavMesh パス探索
  └─ (毎フレーム) PathFollowingComponent->FollowPathSegment()
       └─ CMC->RequestDirectMove() or AddInputVector()
```

### 関与クラス・関数

| クラス | 関数 | 役割 |
|--------|------|------|
| `AAIController` | `OnPossess()` | BT 開始・知覚登録 |
| `AAIController` | `RunBehaviorTree()` | BT アセットの実行開始 |
| `UBehaviorTreeComponent` | `StartTree()` | BT ルートから実行 |
| `UPathFollowingComponent` | `RequestMove()` | ナビメッシュ経路追従 |
| `AAIController` | `SetFocus()` | 優先度付きフォーカス設定 |

---

## 関連ドキュメント

- [[a_player_controller]] — 人間プレイヤー用 Controller
- [[c_possession]] — Possess / UnPossess の詳細フロー
- [[Reference/ref_controller_api]] — AAIController API
