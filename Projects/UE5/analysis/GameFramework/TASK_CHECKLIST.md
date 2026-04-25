# GameFramework ドキュメント チェックリスト

## 概要
- [x] `01_gameframework_overview.md` … GameFramework 全体概要（2026-04-25 完了）
- [x] `_source_map.md` … ソースマップ（2026-04-25 完了）

---

## サブフォルダ構成

### ActorComponent — Actor/Component ライフサイクル
- [x] `ActorComponent/01_overview.md` … ActorComponent 概要（2026-04-25 完了）
- [x] `ActorComponent/Details/a_actor_lifecycle.md` … AActor SpawnActor・BeginPlay・EndPlay・Destroy フロー（2026-04-25）
- [x] `ActorComponent/Details/b_component_model.md` … UActorComponent・USceneComponent・RegisterComponent・Attach（2026-04-25）
- [x] `ActorComponent/Details/c_ticking.md` … FTickFunction・TickGroup・TickDependency・前提条件（2026-04-25）
- [x] `ActorComponent/Details/d_actor_spawning.md` … SpawnActorDeferred・FinishSpawning・テンプレート・ObjectInitializer（2026-04-25）
- [ ] `ActorComponent/Reference/ref_actor_api.md` … AActor 主要 API
- [ ] `ActorComponent/Reference/ref_component_api.md` … UActorComponent / USceneComponent API

### GameModeState — ゲームモード・ゲームステート
- [x] `GameModeState/01_overview.md` … GameModeState 概要（2026-04-25 完了）
- [x] `GameModeState/Details/a_game_mode.md` … AGameModeBase・InitGame・Login・HandleStartingNewPlayer（2026-04-25）
- [x] `GameModeState/Details/b_game_state.md` … AGameStateBase・レプリケーション・MatchState 遷移（2026-04-25）
- [x] `GameModeState/Details/c_game_session.md` … AGameSession・接続承認・OnlineSubsystem 連携（2026-04-25）
- [ ] `GameModeState/Reference/ref_gamemode_api.md` … AGameModeBase / AGameStateBase API

### Controller — コントローラー・ポゼッション
- [x] `Controller/01_overview.md` … Controller 概要（2026-04-25 完了）
- [x] `Controller/Details/a_player_controller.md` … APlayerController・入力処理・HUD・カメラ管理（2026-04-25）
- [x] `Controller/Details/b_ai_controller.md` … AAIController・BrainComponent・RunBehaviorTree（2026-04-25）
- [x] `Controller/Details/c_possession.md` … Possess/UnPossess・ViewTarget・SpectatorPawn（2026-04-25）
- [ ] `Controller/Reference/ref_controller_api.md` … APlayerController / AAIController / APawn API

### CharacterMovement — キャラクター移動
- [x] `CharacterMovement/01_overview.md` … CharacterMovement 概要（2026-04-25 完了）
- [x] `CharacterMovement/Details/a_movement_modes.md` … Walking/Falling/Swimming/Flying・カスタム移動モード（2026-04-25）
- [x] `CharacterMovement/Details/b_cmc_networking.md` … クライアント予測・ServerMove・ClientAdjustment・SavedMove（2026-04-25）
- [x] `CharacterMovement/Details/c_root_motion.md` … RootMotion ソース・AnimMontage/RootMotionTask（2026-04-25）
- [x] `CharacterMovement/Details/d_physics_interaction.md` … 物理インタラクション・StepUp・Floor チェック・Slide（2026-04-25）
- [ ] `CharacterMovement/Reference/ref_cmc_api.md` … UCharacterMovementComponent API
- [ ] `CharacterMovement/Reference/ref_movement_flags.md` … 移動フラグ・EMovementMode 一覧

### Subsystems — サブシステムフレームワーク
- [x] `Subsystems/01_overview.md` … Subsystems 概要（2026-04-25 完了）
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
