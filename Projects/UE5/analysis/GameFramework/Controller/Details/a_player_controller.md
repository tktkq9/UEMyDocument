# APlayerController — プレイヤーコントローラー

- 上位: [[Controller/01_overview]]
- 関連: [[b_ai_controller]] | [[c_possession]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/PlayerController.h`, `Engine/Source/Runtime/Engine/Private/PlayerController.cpp`

---

## 概要

`APlayerController` は人間プレイヤーの意思を Pawn に伝える**ブリッジ**。入力受け取り・カメラ管理・HUD 表示・ネット越しの RPC 送受信を担当する。クライアントとサーバーの両方に存在するが、**入力処理はローカル PC のみ**で動作する。

---

## 主要プロパティ

```cpp
class APlayerController : public AController, public IWorldPartitionStreamingSourceProvider
{
    // ─── 接続 ───────────────────────────────────────────────
    UPROPERTY()
    TObjectPtr<UPlayer>              Player;           // ローカル or ネット接続プレイヤー

    UPROPERTY()
    TObjectPtr<APawn>                AcknowledgedPawn; // クライアントが確認済みの Pawn

    // ─── カメラ ─────────────────────────────────────────────
    UPROPERTY(BlueprintReadOnly, Category=PlayerController)
    TObjectPtr<APlayerCameraManager> PlayerCameraManager;

    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=PlayerController)
    TSubclassOf<APlayerCameraManager> PlayerCameraManagerClass;

    UPROPERTY(EditAnywhere, Category=PlayerController)
    bool bAutoManageActiveCameraTarget;  // Possess 時に自動で ViewTarget を Pawn に設定

    // ─── HUD ────────────────────────────────────────────────
    UPROPERTY()
    TObjectPtr<AHUD> MyHUD;

    // ─── 入力 ────────────────────────────────────────────────
    UPROPERTY(Transient)
    TObjectPtr<UPlayerInput> PlayerInput;  // キー/軸マッピング処理

    UPROPERTY(Transient, BlueprintReadOnly, Category="Cheat Manager")
    TObjectPtr<UCheatManager> CheatManager;
};
```

---

## 入力処理フロー

```
UGameViewportClient::InputKey()
  └─ APlayerController::InputKey()          [PlayerController.cpp]
       └─ UPlayerInput::ProcessInputStack()
            ├─ InputComponent スタック（後入り優先）を走査
            ├─ BindAction / BindAxis マッピングを照合
            └─ 一致したら Delegate.Execute()

APlayerController::PlayerTick()             [PlayerController.cpp:2309]
  ├─ TickPlayerInput()
  │    └─ UPlayerInput::Tick() — 軸値の積算
  ├─ PreProcessInput()
  ├─ ProcessPlayerInput()                   ← BindAxis 系デリゲートここで発火
  └─ PostProcessInput()
```

> `PlayerTick` は `UPlayerInput` を持つ（= ローカル PC の）Controller のみ呼ばれる。

---

## InputComponent スタック

```cpp
// Pawn の SetupPlayerInputComponent に渡してバインド
void AMyPawn::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    PlayerInputComponent->BindAction("Jump", IE_Pressed, this, &AMyPawn::Jump);
    PlayerInputComponent->BindAxis("MoveForward", this, &AMyPawn::MoveForward);
}

// Enhanced Input（UE5 推奨）
void AMyPawn::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
    UEnhancedInputComponent* EIC = Cast<UEnhancedInputComponent>(PlayerInputComponent);
    EIC->BindAction(JumpAction, ETriggerEvent::Started, this, &AMyPawn::Jump);
}
```

スタック管理:
```cpp
// PC が InputComponent を push/pop（UI 表示中に Game 入力を止める など）
PlayerController->PushInputComponent(UIInputComponent);
PlayerController->PopInputComponent(UIInputComponent);
```

---

## 入力モード切り替え

```cpp
// ゲームのみ（デフォルト）
PlayerController->SetInputMode(FInputModeGameOnly());

// UI のみ（カーソル表示・ゲーム入力無効）
FInputModeUIOnly UIMode;
UIMode.SetWidgetToFocus(MyWidget->TakeWidget());
PlayerController->SetInputMode(UIMode);

// ゲーム + UI 両用（インベントリ画面等）
PlayerController->SetInputMode(FInputModeGameAndUI());
```

---

## カメラ管理

```cpp
// ViewTarget を指定 Actor に変更（ブレンド付き）
PlayerController->SetViewTargetWithBlend(
    TargetActor,
    BlendTime=2.0f,
    VTBlend_Cubic,
    BlendExp=2.0f
);

// カメラマネージャー経由でフォーカス変更
PlayerController->PlayerCameraManager->SetFOV(90.f);

// クライアントへ ViewTarget を RPC で送信
// ClientSetViewTarget は UFUNCTION(Client, Reliable)
```

---

## BeginPlayingState — Possess 後の状態遷移

```
APlayerController::OnPossess()           [PlayerController.cpp:877]
  ├─ PawnToPossess->PossessedBy(this)
  ├─ SetControlRotation(Pawn->GetActorRotation())
  ├─ SetPawn(PawnToPossess)
  ├─ ClientRestart(Pawn)               ← クライアントへ RPC
  └─ ChangeState(NAME_Playing)
       └─ BeginPlayingState()           [PlayerController.cpp:5847]
            ├─ SetupInactiveStateInputComponent を無効化
            └─ AcknowledgedPawn を確定
```

---

## カスタム PlayerController の実装例

```cpp
UCLASS()
class AMyPlayerController : public APlayerController
{
    GENERATED_BODY()

public:
    virtual void BeginPlay() override;
    virtual void OnPossess(APawn* InPawn) override;
    virtual void SetupInputComponent() override;

    // スコアボード表示切り替え
    void ToggleScoreboard();

protected:
    UPROPERTY(EditDefaultsOnly)
    TSubclassOf<UUserWidget> ScoreboardWidgetClass;

    UPROPERTY()
    TObjectPtr<UUserWidget> ScoreboardWidget;
};

void AMyPlayerController::SetupInputComponent()
{
    Super::SetupInputComponent();
    // Legacy input
    InputComponent->BindAction("Scoreboard", IE_Pressed, this, &AMyPlayerController::ToggleScoreboard);
}

void AMyPlayerController::OnPossess(APawn* InPawn)
{
    Super::OnPossess(InPawn);
    // カスタム初期化
    if (AMyCharacter* Char = Cast<AMyCharacter>(InPawn))
    {
        Char->SetOwnerController(this);
    }
}
```

---

## ローカル PC か判定

```cpp
// サーバー・クライアント両方に存在するため、処理分岐が必要
if (IsLocalPlayerController())
{
    // HUD 更新などクライアント専用処理
}

if (HasAuthority())
{
    // ゲームルール判定などサーバー専用処理
}
```

---

## 関連 CVar

| CVar | 説明 |
|------|------|
| `input.bEnableGestureRecognizer` | ジェスチャー認識の有効化 |
| `p.NetShowCorrections` | ネット補正デバッグ表示 |
| `showdebug input` | 入力バインド一覧をビューポートに表示 |

---

## コード実行フロー

### 入力処理フロー（ローカル PC のみ）

```
UGameViewportClient::InputKey(Key, Event)            [GameViewportClient.cpp]
  └─ APlayerController::InputKey()                  [PlayerController.cpp]
       └─ UPlayerInput::ProcessInputStack()
            └─ InputComponent スタック（後入り優先）を走査
                 └─ 一致したデリゲートを Execute

APlayerController::PlayerTick(DeltaTime)            [PlayerController.cpp:2309]
  ├─ TickPlayerInput(DeltaTime, bGamePaused)
  │    └─ UPlayerInput::Tick() — 軸値の積算
  ├─ PreProcessInput(DeltaTime, bGamePaused)
  ├─ ProcessPlayerInput(DeltaTime, bGamePaused)     ← BindAxis デリゲート発火
  └─ PostProcessInput(DeltaTime, bGamePaused)
```

### OnPossess → BeginPlayingState

```
APlayerController::OnPossess(Pawn)                  [PlayerController.cpp:877]
  ├─ PawnToPossess->PossessedBy(this)
  ├─ SetControlRotation(Pawn->GetActorRotation())
  ├─ SetPawn(PawnToPossess)
  ├─ CMC->ResetPredictionData_Server()
  ├─ ClientRestart(Pawn)                            ← クライアントへ RPC
  └─ ChangeState(NAME_Playing)
       └─ BeginPlayingState()                       [PlayerController.cpp:5847]
```

### 関与クラス・関数

| クラス | 関数 | 役割 |
|--------|------|------|
| `UGameViewportClient` | `InputKey()` | OS からの入力受信 |
| `UPlayerInput` | `ProcessInputStack()` | InputComponent スタックの走査 |
| `APlayerController` | `PlayerTick()` | 入力処理の毎フレーム更新 |
| `APlayerController` | `OnPossess()` | Pawn 取得後の初期化 |
| `APlayerController` | `BeginPlayingState()` | NAME_Playing 状態への移行 |
| `APlayerController` | `SetViewTargetWithBlend()` | カメラ追従対象の変更 |

---

## 関連ドキュメント

- [[b_ai_controller]] — AI 専用 Controller
- [[c_possession]] — Possess / UnPossess の詳細フロー
- [[../../CharacterMovement/01_overview]] — 入力値を受け取る CMC
- [[Reference/ref_controller_api]] — APlayerController API
