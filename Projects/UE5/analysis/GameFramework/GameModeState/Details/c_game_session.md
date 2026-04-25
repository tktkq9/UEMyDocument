# AGameSession — セッション管理

- 上位: [[GameModeState/01_overview]]
- 関連: [[a_game_mode]] | [[b_game_state]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/GameSession.h`, `Engine/Source/Runtime/Engine/Private/GameSession.cpp`

---

## 概要

`AGameSession` は `AGameModeBase` のサブオブジェクトとして生成される**接続管理レイヤー**。プレイヤーの接続承認・最大人数チェック・キック/BAN などを担当する。`OnlineSubsystem`（Steam / EOS 等）との橋渡しも行う。

> `AGameSession` は `AGameModeBase::GameSessionClass` で指定する。カスタムしない場合は基底実装のままでよい。

---

## 主要クラス構造

```cpp
class AGameSession : public AInfo
{
public:
    // セッション設定
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Game)
    int32 MaxPlayers = 0;           // 0 = 無制限（URL の "?maxplayers" で上書き可）

    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Game)
    int32 MaxSpectators = 2;

    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category=Game)
    bool bRequiresPushToTalk = false;  // ボイスチャット PTT 要否

    // ─── 接続フロー ────────────────────────────────────────────
    // GameModeBase::PreLogin から呼ばれる（ErrorMessage で拒否）
    ENGINE_API virtual void ApproveLogin(const FString& Options, FUniqueNetIdRepl& UniqueId,
                                          APlayerController* NewPlayerController, FString& ErrorMessage);

    // プレイヤーの登録（ログイン承認後）
    ENGINE_API virtual bool RegisterPlayer(APlayerController* NewPlayer,
                                            const FUniqueNetIdRepl& UniqueId, bool bWasFromInvite);

    // プレイヤーの退場通知
    ENGINE_API virtual void UnregisterPlayer(const APlayerController* ExitingPlayer,
                                              const FUniqueNetIdRepl& UniqueId);

    // ─── キック / BAN ───────────────────────────────────────────
    ENGINE_API virtual bool KickPlayer(APlayerController* KickedPlayer, const FText& KickReason);
    ENGINE_API virtual void ReturnToMainMenuHost();

    // ─── マッチ管理 ─────────────────────────────────────────────
    ENGINE_API virtual void HandleMatchHasStarted();
    ENGINE_API virtual void HandleMatchHasEnded();
    ENGINE_API virtual void UpdateSessionJoinability(FName InSessionName, bool bPublicSearchable,
                                                     bool bAllowInvites, bool bJoinViaPresence, ...);
};
```

---

## ApproveLogin フロー

```
新規接続要求
  │
  ▼
UNetDriver::NotifyAcceptingConnection()                [NetDriver.cpp]
  └─ AGameModeBase::PreLogin()                         [GameModeBase.cpp]
       └─ AGameSession::ApproveLogin()                 [GameSession.cpp]
            ├─ if (NumPlayers >= MaxPlayers)
            │    → ErrorMessage = "Server full"        ← 拒否
            ├─ if (bRequiresPassword && !CorrectPwd)
            │    → ErrorMessage = "Incorrect password" ← 拒否
            ├─ OnlineSubsystem checks (BAN 等)
            └─ ErrorMessage.IsEmpty() → 承認
```

---

## 最大接続数の設定

```cpp
// GameSession の設定（プロジェクト設定・URL 引数どちらでも可）
AMyGameSession::AMyGameSession()
{
    MaxPlayers   = 16;   // URL "?maxplayers=X" で上書き可
    MaxSpectators = 4;
}

// GameMode::InitGame でも変更できる
void AMyGameMode::InitGame(const FString& MapName, const FString& Options, FString& ErrorMessage)
{
    Super::InitGame(MapName, Options, ErrorMessage);
    GameSession->MaxPlayers = UGameplayStatics::GetIntOption(Options, TEXT("maxplayers"), 16);
}
```

---

## OnlineSubsystem との連携

```cpp
// オンラインセッションの更新（試合開始時にプライベートにするなど）
void AMyGameSession::HandleMatchHasStarted()
{
    Super::HandleMatchHasStarted();

    IOnlineSubsystem* OnlineSub = IOnlineSubsystem::Get();
    if (OnlineSub)
    {
        IOnlineSessionPtr Sessions = OnlineSub->GetSessionInterface();
        if (Sessions.IsValid())
        {
            // セッションを非公開に（新規参加を締め切る）
            FOnlineSessionSettings Settings;
            Settings.bAllowJoinInProgress = false;
            Sessions->UpdateSession(GameSessionName, Settings, true);
        }
    }
}
```

---

## カスタム GameSession の例

```cpp
// チートコマンドでキック、BANリストを持つカスタム実装
UCLASS()
class AMyGameSession : public AGameSession
{
    GENERATED_BODY()

    TArray<FString> BannedPlayerIds;    // BAN された UniqueId

public:
    virtual void ApproveLogin(const FString& Options, FUniqueNetIdRepl& UniqueId,
                               APlayerController* NewPC, FString& ErrorMessage) override
    {
        // まず基底のチェック（最大人数等）
        Super::ApproveLogin(Options, UniqueId, NewPC, ErrorMessage);
        if (!ErrorMessage.IsEmpty()) return;

        // BAN チェック
        if (BannedPlayerIds.Contains(UniqueId.ToString()))
        {
            ErrorMessage = TEXT("You have been banned from this server.");
        }
    }

    virtual bool KickPlayer(APlayerController* KickedPlayer, const FText& KickReason) override
    {
        bool bResult = Super::KickPlayer(KickedPlayer, KickReason);
        // キックログを記録
        UE_LOG(LogGame, Warning, TEXT("Player kicked: %s - %s"),
               *KickedPlayer->GetPlayerState<APlayerState>()->GetPlayerName(),
               *KickReason.ToString());
        return bResult;
    }
};
```

---

## AGameSession が必要ない場合

単純なシングルプレイや LAN ゲームなら `AGameSession` をカスタムする必要はほとんどない。主にオンラインマッチメイキング・認証・セッション管理が必要な場合に拡張する。

---

## コード実行フロー

### ApproveLogin フロー（詳細）

```
UNetDriver::NotifyAcceptingConnection()              [NetDriver.cpp]
  └─ AGameModeBase::PreLogin()                      [GameModeBase.cpp]
       └─ AGameSession::ApproveLogin(Options, UniqueId, NewPC, ErrorMessage)
            ├─ if (NumPlayers >= MaxPlayers)
            │    ErrorMessage = "Server full"        ← 接続拒否
            ├─ OnlineSubsystem BAN チェック
            └─ ErrorMessage.IsEmpty() == true → 承認

[承認後]
AGameModeBase::Login()
  └─ AGameSession::RegisterPlayer(NewPC, UniqueId, bWasFromInvite)
       └─ (OnlineSubsystem に登録)
```

### HandleMatchHasStarted（セッション非公開化）

```
AGameMode::HandleMatchHasStarted()
  └─ AGameSession::HandleMatchHasStarted()          [GameSession.cpp]
       └─ IOnlineSession::UpdateSession(
               GameSessionName,
               Settings{bAllowJoinInProgress=false},
               true)                                ← 新規参加を締め切り
```

### 関与クラス・関数

| クラス | 関数 | 役割 |
|--------|------|------|
| `AGameSession` | `ApproveLogin()` | 最大人数・BAN チェック |
| `AGameSession` | `RegisterPlayer()` | OnlineSubsystem への登録 |
| `AGameSession` | `UnregisterPlayer()` | 退場時の登録解除 |
| `AGameSession` | `KickPlayer()` | 強制切断 |
| `AGameSession` | `HandleMatchHasStarted()` | 試合開始時のセッション更新 |

---

## 関連ドキュメント

- [[a_game_mode]] — AGameSession を生成・呼び出す AGameModeBase
- [[b_game_state]] — 接続後のプレイヤー状態管理
- [[Reference/ref_gamemode_api]] — AGameSession API
