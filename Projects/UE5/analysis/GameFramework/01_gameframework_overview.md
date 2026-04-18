# Gameplay Framework 全体概要

- 取得対象: `Engine/Source/Runtime/Engine/Classes/GameFramework/`
- 上位: [[UE5 解析インデックス]]

---

## UE5 Gameplay Framework の構成

```
UGameInstance（アプリ全体・シングルトン）
  └─ UWorld
      ├─ AGameModeBase（サーバーのみ）
      │   └─ InitGame / PostLogin / Logout
      ├─ AGameStateBase（全員に複製）
      ├─ APlayerController × N（ローカル / 複製）
      │   ├─ APawn / ACharacter（制御対象）
      │   │   └─ UCharacterMovementComponent
      │   └─ AHUD
      └─ APlayerState × N（全員に複製）
```

| クラス | 役割 |
|--------|------|
| `UGameInstance` | ゲーム全体のシングルトン・セッション管理 |
| `AGameModeBase` | ゲームルール・スポーン・勝利条件 |
| `AGameStateBase` | 全クライアント共有のゲーム状態 |
| `APlayerController` | プレイヤー入力・カメラ・HUD 制御 |
| `APawn` | プレイヤー / AI が制御するアクター基底 |
| `ACharacter` | 歩行キャラ（CharacterMovement 内蔵）|
| `UCharacterMovementComponent` | 移動・ジャンプ・落下・ネット予測 |
| `UGameSubsystem` | GameInstance スコープのサブシステム |
| `UWorldSubsystem` | World スコープのサブシステム |

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `GameInstance.h/.cpp` | ゲームインスタンス |
| `GameModeBase.h/.cpp` | ゲームモード |
| `GameStateBase.h/.cpp` | ゲームステート |
| `PlayerController.h/.cpp` | プレイヤーコントローラー |
| `Character.h/.cpp` | キャラクター |
| `CharacterMovementComponent.h/.cpp` | キャラクター移動 |
| `Pawn.h/.cpp` | ポーン基底 |
| `Subsystems.h/.cpp` | サブシステム基底 |

---

## サブシステムドキュメント

| ドキュメント | 内容 |
|------------|------|
| `Details/a_game_mode_state.md` | GameMode・GameState ライフサイクル |
| `Details/b_player_controller.md` | PlayerController・HUD・カメラ管理 |
| `Details/c_character_movement.md` | CharacterMovement・ネット予測・カスタム移動 |
| `Details/d_pawn_possession.md` | Pawn Possess/UnPossess・AIController との共有 |
| `Details/e_subsystems.md` | GameInstanceSubsystem・WorldSubsystem・ライフサイクル |
