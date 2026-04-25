# AGameStateBase — ゲームステート

- 上位: [[GameModeState/01_overview]]
- 関連: [[a_game_mode]] | [[c_game_session]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/GameStateBase.h`, `GameState.h`, `Engine/Source/Runtime/Engine/Private/GameStateBase.cpp`, `GameState.cpp`

---

## 概要

`AGameStateBase` はゲームの **共有状態**（スコア、残り時間、プレイヤー一覧等）を保持し、**全クライアントに自動複製**される。`AGameModeBase` がサーバ専用なのに対し、`AGameStateBase` はクライアントも参照できる唯一の「ゲームルール情報源」。

---

## GameMode vs GameState の責任分離

```
AGameModeBase（サーバのみ）
  ├─ ルール: "5 キルで勝利"
  ├─ 処理: スポーン / ログイン承認 / 状態遷移
  └─ ストレージ: 不可（複製されない）
                     ↕ 書き込み
AGameStateBase（全員に複製）
  ├─ データ: プレイヤー一覧 / スコア / 残り時間
  ├─ 読み取り: クライアントから可
  └─ 書き込み: サーバのみ
```

---

## 主要クラス構造

```cpp
class AGameStateBase : public AInfo
{
    // ─── 複製プロパティ ──────────────────────────────────────────
    // GameMode クラス（クライアントが型を知るため）
    UPROPERTY(Replicated)
    TSubclassOf<AGameModeBase> GameModeClass;

    // 観戦者 Pawn クラス
    UPROPERTY(ReplicatedUsing=OnRep_SpectatorClass)
    TSubclassOf<ASpectatorPawn> SpectatorClass;

    // BeginPlay が始まったことを複製（クライアント側の BeginPlay を起動）
    UPROPERTY(ReplicatedUsing=OnRep_ReplicatedHasBegunPlay)
    bool bReplicatedHasBegunPlay;

    // サーバの世界時刻（フレーム間の同期用）
    UPROPERTY(ReplicatedUsing=OnRep_ReplicatedWorldTimeSecondsDouble)
    double ReplicatedWorldTimeSecondsDouble;

    // 全プレイヤーの PlayerState 一覧
    UPROPERTY(BlueprintReadOnly, Category=GameState)
    TArray<TObjectPtr<APlayerState>> PlayerArray;

    // ─── プレイヤー管理 ─────────────────────────────────────────
    virtual void AddPlayerState(APlayerState* PlayerState);
    virtual void RemovePlayerState(APlayerState* PlayerState);

    // ─── BeginPlay の起点 ─────────────────────────────────────
    // World BeginPlay 時に AGameModeBase から呼ばれる
    virtual void HandleBeginPlay();

    // サーバの WorldTimeSeconds 取得
    virtual float GetServerWorldTimeSeconds() const;
};
```

---

## AGameState — MatchState の複製

`AGameMode` に対応する拡張版。MatchState をクライアントに複製する:

```cpp
class AGameState : public AGameStateBase
{
    // MatchState を複製（"InProgress" など FName）
    UPROPERTY(ReplicatedUsing=OnRep_MatchState)
    FName MatchState;

    UPROPERTY(BlueprintReadOnly, Category=GameState)
    FName PreviousMatchState;

    UFUNCTION()
    virtual void OnRep_MatchState();

    // 状態クエリ（クライアントから使用可）
    bool HasMatchStarted() const;
    bool IsMatchInProgress() const;
    bool HasMatchEnded() const;
};
```

---

## BeginPlay の起動経路

```
UWorld::BeginPlay()                                      [World.cpp]
  └─ AGameStateBase::HandleBeginPlay()                   [GameStateBase.cpp]
       ├─ bReplicatedHasBegunPlay = true                 ← 複製開始
       ├─ for each Actor in LevelsToBeginPlayedOn:
       │    └─ AActor::BeginPlay()                       ← 全 Actor の BeginPlay
       └─ (クライアント側)
            AGameStateBase::OnRep_ReplicatedHasBegunPlay
              └─ UWorld::BeginPlay() → 同じ経路
```

> クライアントは `bReplicatedHasBegunPlay` の複製を受け取ってから BeginPlay を発火する。接続タイミングによっては `BeginPlay` 済みの Actor が後から届くため、`APlayerController::BeginPlayingState` で対応する。

---

## カスタム GameState の実装例

```cpp
// AMyGameState.h
UCLASS()
class AMyGameState : public AGameState
{
    GENERATED_BODY()

public:
    // 全員に複製されるスコア情報
    UPROPERTY(Replicated, BlueprintReadOnly, Category=Game)
    int32 TeamAScore = 0;

    UPROPERTY(Replicated, BlueprintReadOnly, Category=Game)
    int32 TeamBScore = 0;

    // 残り時間（サーバが毎フレーム更新）
    UPROPERTY(ReplicatedUsing=OnRep_TimeRemaining, BlueprintReadOnly)
    float TimeRemaining = 300.f;

    UFUNCTION()
    void OnRep_TimeRemaining();         // UI に反映等

    // サーバから書き込む関数（クライアントは呼べない）
    void AddScore(int32 TeamIndex, int32 Delta);

protected:
    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
};

// AMyGameState.cpp
void AMyGameState::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(AMyGameState, TeamAScore);
    DOREPLIFETIME(AMyGameState, TeamBScore);
    DOREPLIFETIME(AMyGameState, TimeRemaining);
}

void AMyGameState::OnRep_TimeRemaining()
{
    // UI の残り時間を更新
    if (AMyHUD* HUD = GetWorld()->GetFirstLocalPlayerController()->GetHUD<AMyHUD>())
    {
        HUD->UpdateTimer(TimeRemaining);
    }
}
```

---

## サーバ時刻の同期

```cpp
// クライアントからサーバの時刻を取得
float ServerTime = GetWorld()->GetGameState()->GetServerWorldTimeSeconds();

// ゲームプレイ時間取得（サーバ基準）
float ElapsedTime = ServerTime - GameStartTime;

// アニメーション同期などに使用
// ローカル時刻とのズレ
float TimeDiff = ServerTime - GetWorld()->GetTimeSeconds();
```

---

## APlayerState との関係

`AGameStateBase::PlayerArray` に全プレイヤーの `APlayerState` が格納される:

```cpp
// 全プレイヤーのスコアを列挙
for (APlayerState* PS : GetWorld()->GetGameState()->PlayerArray)
{
    AMyPlayerState* MyPS = Cast<AMyPlayerState>(PS);
    if (MyPS)
    {
        UE_LOG(LogTemp, Log, TEXT("%s: %d kills"), *MyPS->GetPlayerName(), MyPS->Kills);
    }
}
```

---

## 関連 CVar

| CVar | 説明 |
|------|------|
| `net.MaxNetStringSize` | 複製文字列の最大サイズ |
| `net.ReplicationGraphDebugActor` | 複製グラフ デバッグ |

---

## コード実行フロー

### BeginPlay の起動経路

```
UWorld::BeginPlay()                                  [World.cpp]
  └─ AGameStateBase::HandleBeginPlay()              [GameStateBase.cpp]
       ├─ bReplicatedHasBegunPlay = true            ← 複製開始
       └─ for each Actor in Levels:
            └─ AActor::BeginPlay()

[クライアント側]
AGameStateBase::OnRep_ReplicatedHasBegunPlay()       ← 複製受信
  └─ UWorld::BeginPlay()                            → 同じ経路で全 Actor の BeginPlay
```

### PlayerState の登録フロー

```
AGameModeBase::Login() → PostLogin()
  └─ AGameStateBase::AddPlayerState(PlayerState)   [GameStateBase.cpp]
       └─ PlayerArray.AddUnique(PlayerState)        ← 複製プロパティに追加

AGameModeBase::Logout(Exiting)
  └─ AGameStateBase::RemovePlayerState(PlayerState)
       └─ PlayerArray.Remove(PlayerState)
```

### 関与クラス・関数

| クラス | 関数 | 役割 |
|--------|------|------|
| `AGameStateBase` | `HandleBeginPlay()` | World の BeginPlay を全 Actor に伝播 |
| `AGameStateBase` | `OnRep_ReplicatedHasBegunPlay()` | クライアント側 BeginPlay トリガー |
| `AGameStateBase` | `AddPlayerState()` | PlayerArray への登録 |
| `AGameState` | `OnRep_MatchState()` | MatchState 変更をクライアントに通知 |
| `AGameStateBase` | `GetServerWorldTimeSeconds()` | サーバー基準時刻の取得 |

---

## 関連ドキュメント

- [[a_game_mode]] — GameState に書き込む権限側 GameMode
- [[c_game_session]] — 接続承認
- [[Reference/ref_gamemode_api]] — AGameStateBase / AGameState API
- [[../../Network/01_network_overview]] — DOREPLIFETIME / Replicated
