# GameFramework ドキュメント チェックリスト

## 概要
- [ ] `01_gameframework_overview.md` … GameFramework 全体概要（更新予定）
- [ ] `_source_map.md` … ソースマップ

---

## サブフォルダ構成

### ActorComponent — Actor/Component ライフサイクル
- [ ] `ActorComponent/01_overview.md` … ActorComponent 概要
- [ ] `ActorComponent/Details/a_actor_lifecycle.md` … AActor SpawnActor・BeginPlay・EndPlay・Destroy フロー
- [ ] `ActorComponent/Details/b_component_model.md` … UActorComponent・USceneComponent・RegisterComponent・Attach
- [ ] `ActorComponent/Details/c_ticking.md` … FTickFunction・TickGroup・TickDependency・前提条件
- [ ] `ActorComponent/Details/d_actor_spawning.md` … SpawnActorDeferred・FinishSpawning・テンプレート・ObjectInitializer
- [ ] `ActorComponent/Reference/ref_actor_api.md` … AActor 主要 API
- [ ] `ActorComponent/Reference/ref_component_api.md` … UActorComponent / USceneComponent API

### GameModeState — ゲームモード・ゲームステート
- [ ] `GameModeState/01_overview.md` … GameModeState 概要
- [ ] `GameModeState/Details/a_game_mode.md` … AGameModeBase・InitGame・Login・HandleStartingNewPlayer
- [ ] `GameModeState/Details/b_game_state.md` … AGameStateBase・レプリケーション・MatchState 遷移
- [ ] `GameModeState/Details/c_game_session.md` … AGameSession・接続承認・OnlineSubsystem 連携
- [ ] `GameModeState/Reference/ref_gamemode_api.md` … AGameModeBase / AGameStateBase API

### Controller — コントローラー・ポゼッション
- [ ] `Controller/01_overview.md` … Controller 概要
- [ ] `Controller/Details/a_player_controller.md` … APlayerController・入力処理・HUD・カメラ管理
- [ ] `Controller/Details/b_ai_controller.md` … AAIController・BrainComponent・RunBehaviorTree
- [ ] `Controller/Details/c_possession.md` … Possess/UnPossess・ViewTarget・SpectatorPawn
- [ ] `Controller/Reference/ref_controller_api.md` … APlayerController / AAIController / APawn API

### CharacterMovement — キャラクター移動
- [ ] `CharacterMovement/01_overview.md` … CharacterMovement 概要
- [ ] `CharacterMovement/Details/a_movement_modes.md` … Walking/Falling/Swimming/Flying・カスタム移動モード
- [ ] `CharacterMovement/Details/b_cmc_networking.md` … クライアント予測・ServerMove・ClientAdjustment・SavedMove
- [ ] `CharacterMovement/Details/c_root_motion.md` … RootMotion ソース・AnimMontage/RootMotionTask
- [ ] `CharacterMovement/Details/d_physics_interaction.md` … 物理インタラクション・StepUp・Floor チェック・Slide
- [ ] `CharacterMovement/Reference/ref_cmc_api.md` … UCharacterMovementComponent API
- [ ] `CharacterMovement/Reference/ref_movement_flags.md` … 移動フラグ・EMovementMode 一覧

### Subsystems — サブシステムフレームワーク
- [ ] `Subsystems/01_overview.md` … Subsystems 概要
- [ ] `Subsystems/Details/a_subsystem_types.md` … UGameInstanceSubsystem・UWorldSubsystem・ULocalPlayerSubsystem
- [ ] `Subsystems/Details/b_subsystem_lifecycle.md` … Initialize/Deinitialize・ShouldCreateSubsystem・依存関係
- [ ] `Subsystems/Details/c_tickable_subsystem.md` … FTickableGameObject・TickableWorldSubsystem
- [ ] `Subsystems/Reference/ref_subsystem_api.md` … USubsystem / UGameInstanceSubsystem API

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| ActorComponent | 0/1 | 0/4 | 0/2 | 0/7 |
| GameModeState | 0/1 | 0/3 | 0/1 | 0/5 |
| Controller | 0/1 | 0/3 | 0/1 | 0/5 |
| CharacterMovement | 0/1 | 0/4 | 0/2 | 0/7 |
| Subsystems | 0/1 | 0/3 | 0/1 | 0/5 |

**合計**: 概要 1 + ソースマップ 1 + サブフォルダ概要 5 + Details 17 + Reference 7 = **31 ファイル**
