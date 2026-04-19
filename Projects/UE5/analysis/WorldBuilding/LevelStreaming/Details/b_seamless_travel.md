# Seamless / Non-Seamless Travel・TransitionMap

- 上位: [[LevelStreaming/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/GameModeBase.h`
          `Engine/Source/Runtime/Engine/Classes/Engine/Engine.h`

---

## 概要

UE ではレベル（マップ）間の移動に **Seamless Travel** と **Non-Seamless Travel** の 2 種類がある。Seamless はバックグラウンドで新レベルをロードするためプレイヤーが切断されない。

---

## 比較

| 特徴 | Seamless Travel | Non-Seamless Travel |
|------|---------------|-------------------|
| 接続維持 | 維持（クライアント切断なし） | 切断して再接続 |
| ロード方式 | 非同期（TransitionMap 経由） | 同期（ブラック画面） |
| アクタ引き継ぎ | 可能（PlayerController 等） | 不可 |
| 速度 | 遅い（バックグラウンド） | 速い（単純） |
| シングルプレイ | 対応 | 対応 |
| マルチプレイ | 推奨 | 非推奨（切断が必要） |

---

## Seamless Travel の設定

### GameMode での有効化

```cpp
// AGameModeBase のプロパティ
UPROPERTY(EditDefaultsOnly, Category=GameMode)
uint32 bUseSeamlessTravel : 1;  // デフォルト false → true に設定

// Project Settings → Maps & Modes → Transition Map でも設定可
```

### TransitionMap の役割

Seamless Travel は必ず **TransitionMap**（中間マップ）を経由する。

```
現在のマップ → TransitionMap（ローディング画面等）→ 目的地マップ
```

TransitionMap は `Project Settings → Maps & Modes → Transition Map` で設定する。

---

## Travel の実行方法

### サーバー側（`AGameModeBase::ProcessServerTravel`）

```cpp
// URL 形式でレベルを指定
void AMyGameMode::TravelToNextLevel()
{
    // Absolute パスの場合は "/" プレフィックス
    // Seamless の場合は "?listen" を追加（マルチ）
    GetWorld()->ServerTravel(TEXT("/Game/Maps/Level02?listen"), /*bAbsolute=*/false);
}
```

### クライアント側（`APlayerController::ClientTravel`）

```cpp
// シングルプレイでは ClientTravel で直接移動
PlayerController->ClientTravel(TEXT("/Game/Maps/Level02"), TRAVEL_Relative);
```

### `ETravelType` 列挙

```cpp
enum class ETravelType
{
    TRAVEL_Absolute,  // 絶対パスで新マップへ
    TRAVEL_Partial,   // 一部のアクタを保持して移動
    TRAVEL_Relative,  // 現在マップからの相対
};
```

---

## Seamless Travel フロー

```mermaid
sequenceDiagram
    participant GameMode as AGameModeBase (Server)
    participant World as UWorld
    participant Trans as TransitionMap
    participant NewWorld as 新しいマップ

    GameMode->>World: ServerTravel("/Game/Maps/Level02?listen")
    World->>Trans: TransitionMap をロード（非同期）
    Note over World: 古いレベルを徐々にアンロード
    World->>Trans: 遷移完了 → TransitionMap 表示
    World->>NewWorld: 新マップをバックグラウンドでロード
    NewWorld-->>World: ロード完了
    World->>GameMode: PostSeamlessTravel()
    GameMode->>GameMode: InitSeamlessTravelPlayer() 各プレイヤーを再初期化
```

---

## アクタの引き継ぎ

Seamless Travel 中に PlayerController などを新マップに引き継げる。

### サーバー側（GameMode でオーバーライド）

```cpp
// 引き継ぐアクタを追加
virtual void GetSeamlessTravelActorList(
    bool bToTransition,
    TArray<AActor*>& ActorList) override
{
    Super::GetSeamlessTravelActorList(bToTransition, ActorList);

    // TransitionMap への遷移時（bToTransition=true）
    // 新マップへの遷移時（bToTransition=false）
    if (!bToTransition)
    {
        // カスタムアクタを引き継ぎ
        ActorList.Add(MyPersistentActor);
    }
}
```

### クライアント側（PlayerController でオーバーライド）

```cpp
virtual void GetSeamlessTravelActorList(
    bool bToTransition,
    TArray<AActor*>& ActorList) override;
```

---

## Non-Seamless Travel

シンプルな `UGameplayStatics::OpenLevel` / `GetWorld()->ServerTravel` で `bUseSeamlessTravel = false` の場合に実行される。

```cpp
// BP でのレベル遷移（Non-Seamless）
UGameplayStatics::OpenLevel(this, TEXT("/Game/Maps/Level02"));

// C++ での ServerTravel
GetWorld()->ServerTravel(TEXT("/Game/Maps/Level02"), /*bAbsolute=*/true);
```

### Non-Seamless の注意点

- マルチプレイでは全クライアントが切断されるため推奨されない
- ロード画面（BlockingLoad）が必要
- 単純な実装でシングルプレイには十分

---

## FSeamlessTravelHandler

UE 内部では `FSeamlessTravelHandler`（`Engine.h` の内部クラス）が Seamless Travel の状態機械を管理する。

```cpp
// 現在 Seamless Travel 中か確認
bool UWorld::IsInSeamlessTravel() const;

// 移動先 URL の取得
FString UWorld::GetSeamlessTravelDestinationURL() const;
```

---

## 関連イベント

| GameMode コールバック | タイミング |
|-------------------|---------|
| `PostSeamlessTravel()` | 新マップのロード完了後 |
| `HandleSeamlessTravelPlayer(AController*)` | 各プレイヤーを新マップに移行する時 |
| `GetPlayerControllerClassToSpawnForSeamlessTravel(...)` | 新マップで生成するPC クラスを決定 |

---

## コード実行フロー

### エントリポイント

```
[Seamless Travel — サーバーエントリ]
UWorld::ServerTravel(URL, bAbsolute, bShouldSkipGameNotify)
  └─ AGameModeBase::ProcessServerTravel(URL, bAbsolute)
       ├─ bUseSeamlessTravel チェック
       │    ├─ true  → UWorld::SeamlessTravel()
       │    └─ false → UEngine::LoadMap() (Non-Seamless)
       └─ UWorld::SeamlessTravel(URL, bAbsolute)
            └─ NextURL = URL; bRequestedSeamlessTravel = true

[毎フレーム駆動]
UEngine::Tick()
  └─ UEngine::TickWorldTravel(WorldContext, DeltaSeconds)
       └─ if (World->NextURL != "") or SeamlessTravelHandler.IsTraveling():
            └─ FSeamlessTravelHandler::Tick()
                 ├─ Phase1: StartTravel() — TransitionMap ロード開始
                 │    └─ LoadPackageAsync(TransitionMap)
                 │         └─ AGameModeBase::GetSeamlessTravelActorList(bToTransition=true)
                 │              └─ PlayerController/GameState 等を KeepActors に追加
                 ├─ Phase2: TransitionMap 完了
                 │    ├─ 古い World を Tear Down
                 │    ├─ KeepActors を TransitionMap に転送
                 │    └─ AGameModeBase::PostSeamlessTravel() (TransitionMap 側)
                 ├─ Phase3: 目的地マップ非同期ロード
                 │    └─ LoadPackageAsync(DestURL)
                 ├─ Phase4: 目的地完了
                 │    ├─ TransitionMap を Tear Down
                 │    ├─ KeepActors を新マップに転送
                 │    │    └─ GetSeamlessTravelActorList(bToTransition=false)
                 │    ├─ HandleSeamlessTravelPlayer(Controller) — 各プレイヤー再初期化
                 │    └─ AGameModeBase::PostSeamlessTravel() (目的地側)
                 └─ NotifyAllClients → ClientTravel でクライアントも追従

[Non-Seamless Travel]
UEngine::LoadMap(WorldContext, URL, Pending, Error)
  ├─ UEngine::CleanupPackagesToFullyLoad()
  ├─ 全 World を Tear Down（既存コネクション切断）
  ├─ UPackageTools::LoadPackage(URL)  ← 同期ロード
  └─ InitializeNewlyCreatedWorld() → NewWorld->BeginPlay()
```

### フロー詳細

1. **ServerTravel 起点** — サーバー側から `UWorld::ServerTravel()` を呼ぶ。内部で `bUseSeamlessTravel` を確認して Seamless/Non-Seamless を選択。Seamless なら `NextURL` を設定して次フレームの処理に委譲。
2. **TickWorldTravel 駆動** — `UEngine::TickWorldTravel` が毎フレーム `FSeamlessTravelHandler::Tick()` を呼び出し、4 フェーズの状態機械を進行。
3. **Phase1 - TransitionMap ロード** — `GetSeamlessTravelActorList(bToTransition=true)` で引き継ぎアクタを収集。`PlayerController`/`GameState` はデフォルトで含まれる。`LoadPackageAsync` で TransitionMap を非同期ロード。
4. **Phase2 - 中継切替** — TransitionMap ロード完了で古い World を Tear Down し、KeepActors を `Rename()` で TransitionMap に所属替え。`PostSeamlessTravel()` が呼ばれローディング UI を表示できる。
5. **Phase3 - 目的地ロード** — バックグラウンドで目的地マップをロード。TransitionMap が表示され続けるためプレイヤー体験はシームレス。
6. **Phase4 - 最終切替** — 目的地ロード完了で再度 KeepActors を転送（`bToTransition=false`）。`HandleSeamlessTravelPlayer()` が各 `PlayerController` を新マップの GameMode に登録、`InitSeamlessTravelPlayer()` で状態初期化。
7. **クライアント追従** — サーバーが `NotifyAllClients` で `ClientTravel` を送出し、各クライアントが同じ URL に遷移。接続自体は維持されるため再ログイン不要。
8. **Non-Seamless** — `UEngine::LoadMap` は単純。古い World をすべて Tear Down しサーバー/クライアント接続を切断、新マップを同期ロード。マルチでは全員再接続要するので推奨されない。
9. **WP との関係** — Seamless Travel で WP マップへ遷移する場合、`UWorldPartition::Initialize()` が新マップ側の `BeginPlay` 以降で呼ばれる。ストリーミングソース（PlayerController）は引き継がれるため、新マップでも位置が維持される。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `UWorld::ServerTravel` | `World.cpp` | 起点 |
| `UWorld::SeamlessTravel` | `World.cpp` | Seamless フラグ設定 |
| `UEngine::TickWorldTravel` | `UnrealEngine.cpp` | 毎フレーム駆動 |
| `FSeamlessTravelHandler::Tick` | `UnrealEngine.cpp` | フェーズ進行 |
| `AGameModeBase::GetSeamlessTravelActorList` | `GameModeBase.cpp` | 引き継ぎアクタ定義 |
| `AGameModeBase::PostSeamlessTravel` | `GameModeBase.cpp` | 完了コールバック |
| `AGameModeBase::HandleSeamlessTravelPlayer` | `GameModeBase.cpp` | プレイヤー再初期化 |
| `UEngine::LoadMap` | `UnrealEngine.cpp` | Non-Seamless 実装 |
