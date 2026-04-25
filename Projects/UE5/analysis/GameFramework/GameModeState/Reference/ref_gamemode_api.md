# AGameModeBase / AGameMode / AGameStateBase / AGameSession API リファレンス

- 上位: [[GameModeState/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/GameModeBase.h`, `GameMode.h`, `GameStateBase.h`, `GameState.h`, `GameSession.h`

---

## AGameModeBase — クラス設定プロパティ

```cpp
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Classes)
TSubclassOf<APawn>             DefaultPawnClass;
TSubclassOf<APlayerController> PlayerControllerClass;
TSubclassOf<AHUD>              HUDClass;
TSubclassOf<AGameStateBase>    GameStateClass;
TSubclassOf<APlayerState>      PlayerStateClass;
TSubclassOf<AGameSession>      GameSessionClass;

UPROPERTY(BlueprintReadOnly)
FString OptionsString;   // URL パラメータ文字列（?key=val 形式）
```

---

## AGameModeBase — ライフサイクル仮想関数

| 関数 | 説明 |
|------|------|
| `InitGame(MapName, Options, ErrorMessage)` | マップ読み込み後の最初の初期化 |
| `InitGameState()` | GameState Actor の生成・初期化 |
| `PreLogin(Options, Address, UniqueId, ErrorMessage)` | 接続拒否はここで ErrorMessage をセット |
| `Login(NewPlayer, InRemoteRole, Portal, Options, ...)` | PlayerController を生成して返す |
| `PostLogin(NewPlayer)` | ログイン完了後（HUD 送信・データ同期） |
| `HandleStartingNewPlayer(NewPlayer)` | `RestartPlayer` を呼ぶ |
| `Logout(Exiting)` | プレイヤー退場時 |
| `ChoosePlayerStart(Player)` | スポーン地点 APlayerStart を選択 |
| `SpawnDefaultPawnFor(NewPlayer, StartSpot)` | DefaultPawnClass をスポーン |
| `RestartPlayer(NewPlayer)` | スポーン〜Possess の一連処理 |

---

## AGameModeBase — ヘルパー関数

```cpp
// GameState 取得（型安全テンプレート版）
template<class T>
T* GetGameState() const;

AGameStateBase* GameState;  // 直接参照用プロパティ

// スポーン関連
bool PlayerCanRestart(APlayerController* Player);
void SetPlayerDefaults(APawn* PlayerPawn);

// GameSession 取得
AGameSession* GameSession;

// ゲーム終了
void ReturnToMainMenuHost();
```

---

## AGameMode — MatchState 制御

```cpp
// MatchState 定数
namespace MatchState
{
    extern const FName WaitingToStart;   // "WaitingToStart"
    extern const FName InProgress;       // "InProgress"
    extern const FName WaitingPostMatch; // "WaitingPostMatch"
    extern const FName LeavingMap;       // "LeavingMap"
    extern const FName Aborted;          // "Aborted"
}

// 状態遷移
virtual void SetMatchState(FName NewState);
virtual void HandleMatchIsWaitingToStart();
virtual bool ReadyToStartMatch();         // true → WaitingToStart → InProgress
virtual void HandleMatchHasStarted();
virtual bool ReadyToEndMatch();           // true → InProgress → WaitingPostMatch
virtual void HandleMatchHasEnded();
virtual void HandleLeavingMap();
virtual void HandleMatchAborted();

// 強制遷移
void StartMatch();   // WaitingToStart → InProgress
void EndMatch();     // InProgress → WaitingPostMatch
void AbortMatch();
void RestartGame();  // マップリロード

// 状態クエリ
bool HasMatchStarted() const;
bool IsMatchInProgress() const;
bool HasMatchEnded() const;
FName GetMatchState() const;
```

---

## AGameStateBase — 主要プロパティ・関数

```cpp
// 複製プロパティ
UPROPERTY(Replicated)
TSubclassOf<AGameModeBase> GameModeClass;

UPROPERTY(ReplicatedUsing=OnRep_SpectatorClass)
TSubclassOf<ASpectatorPawn> SpectatorClass;

UPROPERTY(ReplicatedUsing=OnRep_ReplicatedHasBegunPlay)
bool bReplicatedHasBegunPlay;

UPROPERTY(ReplicatedUsing=OnRep_ReplicatedWorldTimeSecondsDouble)
double ReplicatedWorldTimeSecondsDouble;

UPROPERTY(BlueprintReadOnly, Category=GameState)
TArray<TObjectPtr<APlayerState>> PlayerArray;

// 関数
virtual void HandleBeginPlay();
virtual float GetServerWorldTimeSeconds() const;
virtual void AddPlayerState(APlayerState* PlayerState);
virtual void RemovePlayerState(APlayerState* PlayerState);
```

---

## AGameState — MatchState 複製

```cpp
UPROPERTY(ReplicatedUsing=OnRep_MatchState)
FName MatchState;

UPROPERTY(BlueprintReadOnly)
FName PreviousMatchState;

// 状態クエリ（クライアントから使用可）
bool HasMatchStarted() const;
bool IsMatchInProgress() const;
bool HasMatchEnded() const;

UFUNCTION()
virtual void OnRep_MatchState();
```

---

## AGameSession — 接続管理

```cpp
// 設定プロパティ
UPROPERTY(EditAnywhere, BlueprintReadOnly)
int32 MaxPlayers     = 0;    // 0 = 無制限
int32 MaxSpectators  = 2;
bool  bRequiresPushToTalk = false;

// 接続フロー
virtual void ApproveLogin(const FString& Options, FUniqueNetIdRepl& UniqueId,
                           APlayerController* NewPC, FString& ErrorMessage);

virtual bool RegisterPlayer(APlayerController* NewPlayer,
                             const FUniqueNetIdRepl& UniqueId, bool bWasFromInvite);

virtual void UnregisterPlayer(const APlayerController* ExitingPlayer,
                               const FUniqueNetIdRepl& UniqueId);

// キック・BAN
virtual bool KickPlayer(APlayerController* KickedPlayer, const FText& KickReason);
virtual void ReturnToMainMenuHost();

// マッチ管理
virtual void HandleMatchHasStarted();
virtual void HandleMatchHasEnded();
virtual void UpdateSessionJoinability(FName InSessionName, bool bPublicSearchable,
                                       bool bAllowInvites, bool bJoinViaPresence, ...);
```

---

## URL オプション解析ユーティリティ

```cpp
// UGameplayStatics ヘルパー（GameMode::InitGame 内で使用）
int32   UGameplayStatics::GetIntOption(const FString& Options, const FString& Key, int32 DefaultValue);
FString UGameplayStatics::ParseOption(const FString& Options, const FString& Key);
bool    UGameplayStatics::HasOption(const FString& Options, const FString& Key);
```

---

## 関連ドキュメント

- [[Details/a_game_mode]] — GameMode ライフサイクル詳細
- [[Details/b_game_state]] — GameState 複製パターン
- [[Details/c_game_session]] — GameSession 接続管理
