# GameFramework ソースマップ

- 対象モジュール: `Engine` (Runtime)
- ヘッダ群:
  - `Engine/Source/Runtime/Engine/Classes/GameFramework/` … Actor / Pawn / Character / GameMode 等のヘッダ
  - `Engine/Source/Runtime/Engine/Classes/Components/` … ActorComponent / SceneComponent 等
  - `Engine/Source/Runtime/Engine/Public/Subsystems/` … Subsystem 系ヘッダ
- 実装群:
  - `Engine/Source/Runtime/Engine/Private/` … Actor.cpp / Pawn.cpp / Character.cpp 等が直下
  - `Engine/Source/Runtime/Engine/Private/Components/` … 各 Component 実装
  - `Engine/Source/Runtime/Engine/Private/Subsystems/` … Subsystem 実装
  - `Engine/Source/Runtime/Engine/Private/GameFramework/` … 周辺機能（SaveGame / RootMotion / SpringArm 等）
- 更新日: 2026-04-25

> Engine モジュールはヘッダ (`Classes/`) と実装 (`Private/`) が分離した古い構造。本ドキュメントでは GameFramework に関連するクラスを **役割別に 5 サブフォルダ**（ActorComponent / GameModeState / Controller / CharacterMovement / Subsystems）に再構成して扱う。

---

## サブフォルダと担当範囲

| サブフォルダ | 主担当 | 概要 |
|------------|--------|------|
| `ActorComponent/` | AActor / UActorComponent / USceneComponent | Actor のスポーン・ティック・破棄・コンポーネント階層 |
| `GameModeState/` | AGameModeBase / AGameStateBase / UGameInstance | ゲームルール・マッチ状態・セッション |
| `Controller/` | AController / APlayerController / AAIController / APawn | コントローラ・ポゼッション・入力受付 |
| `CharacterMovement/` | ACharacter / UCharacterMovementComponent / RootMotion | 歩行/落下/泳ぎ/飛行・ネット予測・カスタム移動 |
| `Subsystems/` | USubsystem / 4 系統サブクラス / FSubsystemCollection | サブシステムフレームワーク |

---

## 主要ヘッダ → クラス対応（GameFramework 直下）

| ファイル | 主要クラス/構造体 | 役割 |
|---------|-----------------|------|
| `Actor.h` | `AActor` | World に配置可能な全アクタの基底 |
| `Pawn.h` | `APawn` | コントローラに憑依される基底 Actor |
| `DefaultPawn.h` | `ADefaultPawn` | 浮遊操作の簡易 Pawn |
| `SpectatorPawn.h` | `ASpectatorPawn` | 観戦用 Pawn |
| `Character.h` | `ACharacter` | CharacterMovement 内蔵の人型キャラ |
| `Controller.h` | `AController` | Pawn を制御する基底クラス |
| `PlayerController.h` | `APlayerController` | ローカル/リモートプレイヤーのコントローラ |
| `PlayerInput.h` | `UPlayerInput` | キーバインド・軸入力管理 |
| `PlayerState.h` | `APlayerState` | プレイヤーごとの複製状態 |
| `PlayerStart.h` | `APlayerStart` | スポーン地点 Actor |
| `PlayerMuteList.h` | `FPlayerMuteList` | ボイスチャット ミュート管理 |
| `GameModeBase.h` | `AGameModeBase` | サーバー専用ゲームルール（ベース） |
| `GameMode.h` | `AGameMode` | MatchState 持ちの拡張 GameMode |
| `GameStateBase.h` | `AGameStateBase` | 全クライアント共有のゲーム状態 |
| `GameState.h` | `AGameState` | MatchState を持つ拡張 GameState |
| `GameSession.h` | `AGameSession` | 接続承認・OnlineSubsystem 連携 |
| `GameNetworkManager.h` | `AGameNetworkManager` | ネット同期パラメータ |
| `HUD.h` | `AHUD` | プレイヤーごとの HUD |
| `CheatManager.h` | `UCheatManager` | デバッグ/チート コマンド |
| `Volume.h` | `AVolume` | トリガ可能なボリューム基底 |
| `PhysicsVolume.h` | `APhysicsVolume` | 物理パラメータを上書きするボリューム |
| `KillZVolume.h` | `AKillZVolume` | 落下死領域 |
| `PainCausingVolume.h` | `APainCausingVolume` | ダメージ ボリューム |
| `CameraBlockingVolume.h` | `ACameraBlockingVolume` | カメラ衝突専用ボリューム |
| `WorldSettings.h` | `AWorldSettings` | ワールド単位の設定（GameMode 上書き等） |
| `Info.h` | `AInfo` | 表示形状を持たない管理用 Actor 基底 |
| `DamageType.h` | `UDamageType` | ダメージ分類 |
| `MovementComponent.h` | `UMovementComponent` | 移動コンポーネント基底 |
| `PawnMovementComponent.h` | `UPawnMovementComponent` | Pawn 用 移動コンポーネント |
| `CharacterMovementComponent.h` | `UCharacterMovementComponent` | Character 用本格的移動 |
| `CharacterMovementReplication.h` | `FCharacterMoveResponseDataContainer` 等 | CMC ネット予測の構造体 |
| `RootMotionSource.h` | `FRootMotionSource` 等 | ルートモーション ソース基底 |
| `FloatingPawnMovement.h` | `UFloatingPawnMovement` | 浮遊型 Pawn 移動 |
| `ProjectileMovementComponent.h` | `UProjectileMovementComponent` | 弾道移動 |
| `RotatingMovementComponent.h` | `URotatingMovementComponent` | 自動回転 |
| `NavMovementComponent.h` | `UNavMovementComponent` | NavMesh と連動する移動 |
| `SpringArmComponent.h` | `USpringArmComponent` | TPS カメラ用スプリング アーム |
| `SaveGame.h` | `USaveGame` | SaveGame 基底クラス |
| `AsyncActionHandleSaveGame.h` | `UAsyncActionHandleSaveGame` | BP 非同期セーブ/ロード |
| `InputDeviceSubsystem.h` | `UInputDeviceSubsystem` | 入力デバイス管理サブシステム |
| `InputSettings.h` | `UInputSettings` | プロジェクト入力設定 |
| `LocalMessage.h` | `ULocalMessage` | ローカル メッセージ基底 |
| `OnlineSession.h` | `UOnlineSession` | OnlineSubsystem の薄いラッパ |
| `OnlineReplStructs.h` | `FUniqueNetIdRepl` | ネット ID の複製可能構造体 |
| `LightWeightInstanceManager.h` | `ALightWeightInstanceManager` | 軽量インスタンス管理 |
| `ForceFeedbackEffect.h` | `UForceFeedbackEffect` | コントローラ振動 |

---

## 主要ヘッダ → クラス対応（Components/）

| ファイル | 主要クラス | 役割 |
|---------|-----------|------|
| `ActorComponent.h` | `UActorComponent` | 全コンポーネントの基底 |
| `SceneComponent.h` | `USceneComponent` | Transform を持つコンポーネント |
| `PrimitiveComponent.h` | `UPrimitiveComponent` | レンダ/コリジョンを持つコンポーネント |
| `InputComponent.h` | `UInputComponent` | 入力バインディング |

---

## 主要ヘッダ → クラス対応（Subsystems/）

| ファイル | 主要クラス | 役割 |
|---------|-----------|------|
| `Subsystem.h` | `USubsystem` | 全サブシステムの基底（`UObject` 派生） |
| `EngineSubsystem.h` | `UEngineSubsystem` | `UEngine` スコープ |
| `GameInstanceSubsystem.h` | `UGameInstanceSubsystem` | `UGameInstance` スコープ |
| `WorldSubsystem.h` | `UWorldSubsystem` / `UTickableWorldSubsystem` | `UWorld` スコープ |
| `LocalPlayerSubsystem.h` | `ULocalPlayerSubsystem` | `ULocalPlayer` スコープ |
| `AudioEngineSubsystem.h` | `UAudioEngineSubsystem` | オーディオエンジン スコープ |
| `SubsystemCollection.h` | `FSubsystemCollection<T>` / `FSubsystemCollectionBase` | サブシステム コレクション本体 |
| `SubsystemBlueprintLibrary.h` | `USubsystemBlueprintLibrary` | BP からのサブシステム取得 |

---

## 主要 .cpp の場所

| クラス | 実装ファイル |
|--------|-----------|
| `AActor` | `Engine/Source/Runtime/Engine/Private/Actor.cpp` |
| `APawn` | `Private/Pawn.cpp` |
| `ACharacter` | `Private/Character.cpp` |
| `AController` | `Private/Controller.cpp` |
| `APlayerController` | `Private/PlayerController.cpp` |
| `UPlayerInput` | `Private/UserInterface/PlayerInput.cpp` |
| `APlayerState` | `Private/PlayerState.cpp` |
| `AGameModeBase` / `AGameMode` | `Private/GameModeBase.cpp` / `Private/GameMode.cpp` |
| `AGameStateBase` / `AGameState` | `Private/GameStateBase.cpp` / `Private/GameState.cpp` |
| `AGameSession` | `Private/GameSession.cpp` |
| `UGameInstance` | `Private/GameInstance.cpp` |
| `AHUD` | `Private/HUD.cpp` |
| `UCheatManager` | `Private/CheatManager.cpp` |
| `AWorldSettings` | `Private/WorldSettings.cpp` |
| `UActorComponent` | `Private/Components/ActorComponent.cpp` |
| `USceneComponent` | `Private/Components/SceneComponent.cpp` |
| `UMovementComponent` | `Private/Components/MovementComponent.cpp` |
| `UPawnMovementComponent` | `Private/Components/PawnMovementComponent.cpp` |
| `UCharacterMovementComponent` | `Private/Components/CharacterMovementComponent.cpp` |
| `URotatingMovementComponent` 他 | `Private/Components/*.cpp` |
| `USubsystem` 系 | `Private/Subsystems/*.cpp` |
| `FSubsystemCollection` | `Private/Subsystems/SubsystemCollection.cpp` |
| `USpringArmComponent` | `Private/GameFramework/SpringArmComponent.cpp` |
| `FRootMotionSource` | `Private/GameFramework/RootMotionSource.cpp` |
| `USaveGame` | `Private/GameFramework/SaveGame.cpp` |
| `UInputDeviceSubsystem` | `Private/GameFramework/InputDeviceSubsystem.cpp` |

---

## エントリポイント（主要関数）

### Actor / Component ライフサイクル

| 関数 | ファイル | 説明 |
|------|---------|------|
| `UWorld::SpawnActor<T>` | `Private/LevelActor.cpp` | テンプレート版 SpawnActor |
| `UWorld::SpawnActorAbsolute` / `SpawnActorDeferred` | `Private/LevelActor.cpp` | 配置・遅延スポーン |
| `AActor::PostSpawnInitialize` | `Private/Actor.cpp` | スポーン直後の初期化 |
| `AActor::FinishSpawning` | `Private/Actor.cpp` | DeferredSpawn 完了 |
| `AActor::PreInitializeComponents` / `PostInitializeComponents` | `Private/Actor.cpp` | コンポーネント初期化 |
| `AActor::BeginPlay` | `Private/Actor.cpp` | プレイ開始通知 |
| `AActor::EndPlay` | `Private/Actor.cpp` | プレイ終了通知 |
| `AActor::Tick` | `Private/Actor.cpp` | フレーム ティック |
| `AActor::Destroy` / `K2_DestroyActor` | `Private/Actor.cpp` | 破棄要求 |
| `UActorComponent::RegisterComponent` | `Private/Components/ActorComponent.cpp` | コンポーネント登録 |
| `USceneComponent::AttachToComponent` | `Private/Components/SceneComponent.cpp` | アタッチ |

### GameMode / GameState

| 関数 | ファイル | 説明 |
|------|---------|------|
| `AGameModeBase::InitGame` | `Private/GameModeBase.cpp` | ゲーム初期化 |
| `AGameModeBase::PreLogin` / `PostLogin` | `Private/GameModeBase.cpp` | プレイヤー接続承認 |
| `AGameModeBase::HandleStartingNewPlayer` | `Private/GameModeBase.cpp` | 新規プレイヤー処理 |
| `AGameModeBase::SpawnDefaultPawnFor` | `Private/GameModeBase.cpp` | Pawn スポーン |
| `AGameModeBase::Logout` | `Private/GameModeBase.cpp` | プレイヤー退出 |
| `AGameMode::SetMatchState` | `Private/GameMode.cpp` | MatchState 遷移 |
| `AGameStateBase::HandleBeginPlay` | `Private/GameStateBase.cpp` | 全 Actor BeginPlay |

### Controller / Pawn

| 関数 | ファイル | 説明 |
|------|---------|------|
| `AController::Possess` / `UnPossess` | `Private/Controller.cpp` | Pawn 憑依 |
| `APlayerController::SetupInputComponent` | `Private/PlayerController.cpp` | 入力バインド |
| `APlayerController::PlayerTick` | `Private/PlayerController.cpp` | プレイヤー ティック |
| `APlayerController::ProcessPlayerInput` | `Private/PlayerController.cpp` | 入力処理 |
| `APawn::SetupPlayerInputComponent` | `Private/Pawn.cpp` | Pawn 側入力バインド |
| `APawn::PossessedBy` / `UnPossessed` | `Private/Pawn.cpp` | 憑依通知 |

### CharacterMovement

| 関数 | ファイル | 説明 |
|------|---------|------|
| `UCharacterMovementComponent::TickComponent` | `Private/Components/CharacterMovementComponent.cpp` | 移動メイン ループ |
| `UCharacterMovementComponent::PerformMovement` | 同上 | フレームの移動本体 |
| `UCharacterMovementComponent::PhysWalking` 等 | 同上 | モード別物理計算 |
| `UCharacterMovementComponent::ServerMove_Implementation` | 同上 | クライアント→サーバ移動 |
| `UCharacterMovementComponent::ClientAdjustPosition_Implementation` | 同上 | サーバ→クライアント補正 |
| `UCharacterMovementComponent::ReplicateMoveToServer` | 同上 | SavedMove 構築 |

### Subsystems

| 関数 | ファイル | 説明 |
|------|---------|------|
| `FSubsystemCollectionBase::Initialize` | `Private/Subsystems/SubsystemCollection.cpp` | コレクション初期化 |
| `FSubsystemCollectionBase::AddAndInitializeSubsystem` | 同上 | サブシステム追加 |
| `USubsystem::Initialize` / `Deinitialize` | `Private/Subsystems/Subsystem.cpp` | 仮想ライフサイクル |
| `USubsystem::ShouldCreateSubsystem` | 同上 | 生成判定 |
| `UEngine::Init` / `UGameInstance::Init` / `UWorld::PostInitializeComponents` | `Private/UnrealEngine.cpp` 等 | サブシステム生成タイミング |

---

## 主要マクロ

| マクロ | 役割 |
|--------|------|
| `AActor*` | UCLASS で `Spawnable` 等の指定子付き |
| `ServerMove` 等 | `UFUNCTION(Server, Reliable, WithValidation)` |
| `ClientAdjustPosition` 等 | `UFUNCTION(Client, Reliable)` |
| `GENERATED_BODY()` | UHT がリフレクション コードを生成 |
| `UPROPERTY(Replicated)` | レプリケーション対象プロパティ |
| `UPROPERTY(BlueprintReadOnly)` 等 | BP 公開 |

---

## 主要 CVar

| CVar | 説明 |
|------|------|
| `p.NetEnableMoveCombining` | ServerMove の結合 |
| `p.NetEnableMoveCombiningOnStaticBaseChange` | 静的 Base 変化時の結合 |
| `t.MaxFPS` | フレームレート制限（Ticking 影響） |
| `gc.MaxObjectsNotConsideredByGC` | GC 連動 |
| `net.AllowAsyncLoading` | 非同期ロード |

---

## ナビゲーション

| やりたいこと | 開始ファイル |
|------------|-----------|
| Actor のスポーン手順を追う | `Private/LevelActor.cpp::SpawnActor` → `Private/Actor.cpp::PostSpawnInitialize` |
| BeginPlay 経路を追う | `UWorld::BeginPlay` → `AGameStateBase::HandleBeginPlay` → `AActor::BeginPlay` |
| プレイヤー接続を追う | `AGameModeBase::PreLogin` → `PostLogin` → `HandleStartingNewPlayer` → `RestartPlayer` |
| 移動同期を追う | `UCharacterMovementComponent::ReplicateMoveToServer` → `ServerMove_Implementation` |
| サブシステム生成を追う | `UGameInstance::Init` → `FSubsystemCollection::Initialize` → `AddAndInitializeSubsystem` |

---

## 関連システム

- **Core/UObject** — `AActor` は `UObject` 派生（[[../Core/UObject/01_overview]]）
- **Network** — Pawn / Character の同期は `UNetDriver` 経由（[[../Network/01_network_overview]]）
- **AI** — `AAIController` は別系統（[[../AI/01_ai_overview]]）
- **Input** — `UPlayerInput` は EnhancedInput と協働（[[../Input/01_input_overview]]）
- **Animation** — `ACharacter::Mesh` は `USkeletalMeshComponent`（[[../Animation/01_animation_overview]]）
