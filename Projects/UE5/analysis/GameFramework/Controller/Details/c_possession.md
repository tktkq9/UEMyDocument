# Possess / UnPossess — ポゼッションフロー

- 上位: [[Controller/01_overview]]
- 関連: [[a_player_controller]] | [[b_ai_controller]]
- ソース: `Engine/Source/Runtime/Engine/Private/Controller.cpp`, `Pawn.cpp`, `PlayerController.cpp`

---

## 概要

「ポゼッション（Possession）」は Controller が Pawn を制御下に置く操作。`AController::Possess()` を呼ぶとサーバー権限チェック → Pawn 側コールバック → Controller 側通知の順に処理が走り、ネット越しではクライアントに `ClientRestart` RPC が送られる。

> UE4.22 以降、`Possess()` / `UnPossess()` は `final` になりオーバーライド不可。カスタム処理は `OnPossess()` / `OnUnPossess()` を使う。

---

## Possess フロー（サーバー）

```
AController::Possess(InPawn)              [Controller.cpp:316]
  ├─ bCanPossessWithoutAuthority チェック（権限がないと Warning + return）
  ├─ OnPossess(InPawn)                    ← virtual → 派生でオーバーライド
  │    ├─ (既存の Pawn があれば UnPossess())
  │    ├─ InPawn->PossessedBy(this)       [Pawn.cpp:650]
  │    │    ├─ SetOwner(NewController)
  │    │    ├─ SetController(NewController)
  │    │    ├─ ForceNetUpdate()
  │    │    ├─ UpdateOwningNetConnection()
  │    │    ├─ PlayerState → SetPlayerState()
  │    │    ├─ (PlayerController の場合) SetAutonomousProxy(true)
  │    │    └─ ReceivePossessed() — Blueprint イベント
  │    ├─ SetPawn(InPawn)
  │    ├─ SetControlRotation(Pawn->GetActorRotation())
  │    └─ Pawn->DispatchRestart(false)
  ├─ ReceivePossess(NewPawn)              — Blueprint デリゲート
  └─ OnNewPawn.Broadcast(NewPawn)         — C++ デリゲート
```

### APlayerController::OnPossess の追加処理

```
APlayerController::OnPossess(PawnToPossess)  [PlayerController.cpp:877]
  ├─ Super::OnPossess() — 基本フロー（上記）
  ├─ CMC の ResetPredictionData_Server()    — ネット予測リセット
  ├─ ClientRestart(Pawn)                    — クライアントへ RPC 送信
  └─ ChangeState(NAME_Playing)
       ├─ BeginPlayingState()
       └─ bAutoManageActiveCameraTarget なら AutoManageActiveCameraTarget(Pawn)
```

---

## クライアント側の Possess 確認フロー

```
[Server] APlayerController::ClientRestart(Pawn)
  ↓ (RPC 受信)
[Client] APlayerController::ClientRestart_Implementation()
  └─ Pawn->PawnClientRestart()               [Pawn.cpp:475]
       ├─ Pawn->Restart()                    ← BeginPlay 後のリセット処理
       ├─ SetupPlayerInputComponent()         ← InputComponent を生成してバインド
       └─ EnableInput(this)

[Client] APlayerController::AcknowledgePossession(Pawn)
  └─ AcknowledgedPawn = Pawn                 ← サーバーへ ServerAcknowledgePossession RPC
```

`AcknowledgedPawn` はサーバーがクライアントの "見ている Pawn" を追跡するための参照。一致しない間はクライアント側の移動入力が破棄される。

---

## UnPossess フロー

```
AController::UnPossess()                  [Controller.cpp:382]
  └─ OnUnPossess()                        ← virtual
       ├─ Pawn->UnPossessed()             [Pawn.cpp:692]
       │    ├─ ForceNetUpdate()
       │    ├─ SetPlayerState(nullptr)
       │    ├─ SetOwner(nullptr)
       │    ├─ SetController(nullptr)
       │    ├─ DestroyPlayerInputComponent()
       │    └─ ReceiveUnPossessed() — Blueprint イベント
       └─ SetPawn(NULL)
```

Pawn 破棄時 (`PawnPendingDestroy`) は自動で `UnPossess()` → `ChangeState(NAME_Inactive)` → PlayerState が null なら `Destroy()`。

---

## SpectatorPawn — 観戦者モード

```cpp
// PlayerController が Spectator になるフロー
APlayerController::UnPossess()
  └─ ChangeState(NAME_Spectating)
       └─ BeginSpectatingState()
            └─ SpawnSpectatorPawn()  ← DefaultSpectatorClass からスポーン
                 └─ Possess(SpectatorPawn)
```

```cpp
// カスタム Spectator 制御
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=PlayerController)
TSubclassOf<ASpectatorPawn> SpectatorClass;

// 観戦状態チェック
if (PlayerController->IsInState(NAME_Spectating))
{
    ASpectatorPawn* Spec = PlayerController->GetSpectatorPawn();
}
```

---

## ViewTarget — カメラ追従対象

ViewTarget は Pawn と独立して管理される（観戦カメラ等で別 Actor を見せる場合に使用）:

```cpp
// ViewTarget を変更
PlayerController->SetViewTargetWithBlend(
    NewTarget,          // 見せたい Actor
    BlendTime=1.5f,
    VTBlend_EaseInOut,
    BlendExp=2.f
);

// 現在の ViewTarget を取得
AActor* VT = PlayerController->GetViewTarget();

// Pawn への ViewTarget コールバック
// AActor::BecomeViewTarget(APlayerController* PC)
// AActor::EndViewTarget(APlayerController* PC)
```

---

## カスタム OnPossess の実装例

```cpp
// AMyAIController.cpp
void AMyAIController::OnPossess(APawn* InPawn)
{
    Super::OnPossess(InPawn);

    // AI Logic は Super 内の bStartAILogicOnPossess が true なら自動起動
    // ここでは追加の初期化
    if (AMyCharacter* Char = Cast<AMyCharacter>(InPawn))
    {
        Char->OnDeathDelegate.AddUObject(this, &AMyAIController::OnPawnDeath);
    }
}

void AMyAIController::OnUnPossess()
{
    if (AMyCharacter* Char = Cast<AMyCharacter>(GetPawn()))
    {
        Char->OnDeathDelegate.RemoveAll(this);
    }
    Super::OnUnPossess();
}
```

---

## ネット環境での注意点

| 状況 | 対応 |
|------|------|
| `Possess()` はサーバーのみ呼ぶ | クライアントから呼ぶと Warning + 無視 |
| `AcknowledgedPawn` が一致するまで | クライアントの入力は捨てられる |
| `SetAutonomousProxy(true)` | PlayerController が Possess した Pawn は自律プロキシに昇格 |
| AI ポゼッション | `SimulatedProxy` のまま（入力なし）、移動はサーバー計算 |

---

## 関連ドキュメント

- [[a_player_controller]] — PlayerController が持つ入力・カメラ管理
- [[b_ai_controller]] — AIController の BT / 知覚フロー
- [[../../ActorComponent/Details/a_actor_lifecycle]] — Pawn の BeginPlay との関係
- [[Reference/ref_controller_api]] — AController / APawn API
