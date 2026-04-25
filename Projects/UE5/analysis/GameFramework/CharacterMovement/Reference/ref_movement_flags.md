# 移動フラグ・EMovementMode 一覧

- 上位: [[CharacterMovement/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/CharacterMovementComponent.h`, `EngineTypes.h`

---

## EMovementMode

```cpp
UENUM(BlueprintType)
enum EMovementMode : int
{
    MOVE_None        = 0,   // 移動なし（ラグドール・アニメーション制御中）
    MOVE_Walking     = 1,   // 地面歩行（FindFloor + PhysWalking）
    MOVE_NavWalking  = 2,   // NavMesh 上歩行（軽量・低精度）
    MOVE_Falling     = 3,   // 落下・ジャンプ中（PhysFalling）
    MOVE_Swimming    = 4,   // 水中（PhysSwimming）
    MOVE_Flying      = 5,   // 飛行・重力なし（PhysFlying）
    MOVE_Custom      = 6,   // カスタム（PhysCustom + CustomMovementMode）
    MOVE_MAX         = 7,
};
```

---

## CompressedFlags（ネット同期ビットフィールド）

```cpp
// FSavedMove_Character::GetCompressedFlags() で使用
enum CompressedFlags : uint8
{
    FLAG_JumpPressed     = 0x01,  // bPressedJump
    FLAG_WantsToCrouch   = 0x02,  // bWantsToCrouch
    FLAG_Reserved_1      = 0x04,  // 予約済み
    FLAG_Reserved_2      = 0x08,  // 予約済み
    FLAG_Custom_0        = 0x10,  // カスタム入力 0（スプリント等）
    FLAG_Custom_1        = 0x20,  // カスタム入力 1
    FLAG_Custom_2        = 0x40,  // カスタム入力 2
    FLAG_Custom_3        = 0x80,  // カスタム入力 3
};
```

---

## EPathFollowingRequestResult（AAIController::MoveToActor 戻り値）

```cpp
namespace EPathFollowingRequestResult
{
    enum Type
    {
        Failed,           // パス探索失敗（ナビメッシュなし等）
        AlreadyAtGoal,    // 既に目標位置にいる
        RequestSuccessful,// パス探索要求成功（移動開始）
    };
}
```

---

## EPathFollowingStatus（移動状態）

```cpp
namespace EPathFollowingStatus
{
    enum Type
    {
        Idle,      // 移動していない
        Waiting,   // パス計算待ち
        Paused,    // 一時停止中
        Moving,    // 移動中
    };
}
```

---

## EAIFocusPriority（フォーカス優先度）

```cpp
namespace EAIFocusPriority
{
    enum Type
    {
        Default  = 0,   // 最低優先
        Move     = 1,   // 移動中の自動フォーカス
        Gameplay = 2,   // ゲームプレイロジックによるフォーカス（最高優先）
    };
}
```

---

## ENetworkSmoothingMode（SimulatedProxy 補間）

```cpp
UENUM()
enum class ENetworkSmoothingMode : uint8
{
    Disabled,     // 補間なし（デバッグ用）
    Linear,       // 線形補間（デフォルト）
    Exponential,  // 指数補間（滑らか）
    Replay,       // リプレイ再生専用
};
```

---

## ETeleportType（位置設定時の挙動）

```cpp
UENUM()
enum class ETeleportType : uint8
{
    None,              // 通常移動（物理・スイープ適用）
    TeleportPhysics,   // 物理を維持しつつ瞬間移動
    ResetPhysics,      // 物理状態もリセット
};
```

---

## ESpawnActorCollisionHandlingMethod（スポーン時コリジョン処理）

```cpp
UENUM(BlueprintType)
enum class ESpawnActorCollisionHandlingMethod : uint8
{
    Undefined,                          // デフォルト（プロジェクト設定に従う）
    AlwaysSpawn,                        // コリジョン無視して常にスポーン
    AdjustIfPossibleButAlwaysSpawn,     // 可能なら位置調整してスポーン
    AdjustIfPossibleButDontSpawnIfColliding, // 位置調整失敗ならスポーンしない
    DontSpawnIfColliding,               // コリジョンがあればスポーンしない
};
```

---

## EEndPlayReason（EndPlay の理由）

```cpp
UENUM(BlueprintType)
enum class EEndPlayReason : uint8
{
    Destroyed,           // Destroy() が呼ばれた
    WorldTransition,     // レベル遷移
    EndPlayInEditor,     // エディタの PIE 終了
    RemovedFromWorld,    // Streaming で Level がアンロード
    Quit,                // アプリ終了
};
```

---

## ETickingGroup（Tick 実行グループ）

```cpp
UENUM()
enum ETickingGroup : int
{
    TG_PrePhysics          = 0,   // 物理シミュレーション前
    TG_StartPhysics,              // 物理開始（非同期開始用）
    TG_DuringPhysics,             // 物理計算と並行
    TG_EndPhysics,                // 物理終了（非同期完了待ち）
    TG_PostPhysics,               // 物理シミュレーション後
    TG_PostUpdateWork,            // カメラ等の後処理
    TG_LastDemotable,             // 最後に実行（降格可能）
    TG_NewlySpawned,              // スポーン直後の初回 Tick
    TG_MAX
};
```

---

## EAttachmentRule / FAttachmentTransformRules

```cpp
UENUM()
enum class EAttachmentRule : uint8
{
    KeepRelative,  // 相対 Transform を維持
    KeepWorld,     // ワールド Transform を維持
    SnapToTarget,  // 親のソケットにスナップ（Transform ゼロ）
};

// よく使う組み合わせ定数
FAttachmentTransformRules::KeepRelativeTransform   // 全軸 KeepRelative
FAttachmentTransformRules::KeepWorldTransform      // 全軸 KeepWorld
FAttachmentTransformRules::SnapToTargetNotIncludingScale
FAttachmentTransformRules::SnapToTargetIncludingScale
```

---

## 関連ドキュメント

- [[Details/a_movement_modes]] — 各 EMovementMode の動作
- [[Details/b_cmc_networking]] — CompressedFlags の活用
- [[ref_cmc_api]] — UCharacterMovementComponent 全 API
