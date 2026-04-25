# CMC ネットワーク予測 — SavedMove / ServerMove / ClientAdjustPosition

- 上位: [[CharacterMovement/01_overview]]
- 関連: [[a_movement_modes]] | [[c_root_motion]]
- ソース: `Engine/Source/Runtime/Engine/Private/Components/CharacterMovementComponent.cpp`

---

## 概要

`UCharacterMovementComponent` のネット予測は **クライアント権威モデル** に近い設計：

1. クライアントがローカルで移動を実行し `SavedMove` に保存
2. サーバーへ `ServerMove` RPC で送信
3. サーバーが再計算し、ズレが閾値を超えたら `ClientAdjustPosition` で補正

この仕組みにより、入力遅延なしの滑らか操作と位置の整合性が両立する。

---

## 全体フロー

```
[クライアント – 毎フレーム]
TickComponent()
  └─ PerformMovement()               ← ローカル物理を即時実行（ラグなし表示）
  └─ ReplicateMoveToServer()         [CMC.cpp:8789]
       ├─ FSavedMove_Character に現在状態を記録
       ├─ PendingMove があれば 2 手まとめて送信（Move Combining）
       └─ ServerMovePacked_ClientSend() — RPC 送信

[サーバー – RPC 受信]
ServerMove_PerformMovement()         [CMC.cpp:9840]
  ├─ MoveData を展開して PerformMovement() を実行
  └─ ServerMoveHandleClientError()   [CMC.cpp:10051]
       ├─ クライアント位置 vs サーバー位置を比較
       ├─ ズレ < MAXPOSITIONERRORSQUARED → ACK のみ
       └─ ズレ超過 → ClientAdjustPosition() RPC

[クライアント – 補正受信]
ClientAdjustPosition_Implementation() [CMC.cpp:11052]
  ├─ サーバー位置に瞬間移動
  └─ ClientUpdatePositionAfterServerUpdate() [CMC.cpp:8488]
       └─ SavedMoves を全て再生（Replay）してローカル予測を再適用
```

---

## FSavedMove_Character

SavedMove はクライアントが「まだサーバーに確認されていない移動」を保持するリング:

```cpp
class FSavedMove_Character
{
    float     TimeStamp;        // 移動のタイムスタンプ
    float     DeltaTime;
    FVector   Acceleration;     // 入力加速度
    FVector   SavedLocation;    // 移動前位置
    FQuat     SavedRotation;
    FVector   SavedVelocity;
    uint8     SavedMovementMode;

    // 入力フラグ（ジャンプ、しゃがみ等）
    uint8     CompressedFlags;  // bPressedJump, bWantsToCrouch 等をビットパック

    // ── オーバーライドポイント ──────────────────────────────
    virtual void SetInitialPosition(ACharacter* C);  // 保存
    virtual void PrepMoveFor(ACharacter* C);          // 再生準備
    virtual bool CanCombineWith(const FSavedMovePtr& NewMove, ACharacter* C, float MaxDelta) const;
    virtual uint8 GetCompressedFlags() const;
};
```

### Move Combining

同一フレームで 2 回送信するコストを抑えるため、連続する似た移動をまとめる:

```cpp
// CanCombineWith が true なら 1 RPC に合算
// デフォルト: AccelDotThreshold > 0.99 かつ同じ MovementMode のとき結合可
```

---

## CompressedFlags — 入力ビットパック

```cpp
// デフォルト実装
uint8 FSavedMove_Character::GetCompressedFlags() const
{
    uint8 Result = 0;
    if (bPressedJump)   Result |= FLAG_JumpPressed;
    if (bWantsToCrouch) Result |= FLAG_WantsToCrouch;
    // FLAG_Custom_0 〜 FLAG_Custom_3 はカスタム入力用に予約
    return Result;
}
```

カスタム入力（スプリント等）をネット同期する場合:

```cpp
class FMySavedMove : public FSavedMove_Character
{
    uint8 bWantsToSprint : 1;

    virtual void SetInitialPosition(ACharacter* C) override
    {
        Super::SetInitialPosition(C);
        bWantsToSprint = Cast<AMyCharacter>(C)->bWantsToSprint;
    }

    virtual uint8 GetCompressedFlags() const override
    {
        uint8 Flags = Super::GetCompressedFlags();
        if (bWantsToSprint) Flags |= FLAG_Custom_0;
        return Flags;
    }

    virtual void PrepMoveFor(ACharacter* C) override
    {
        Super::PrepMoveFor(C);
        Cast<AMyCharacter>(C)->bWantsToSprint = bWantsToSprint;
    }
};
```

---

## サーバーの補正処理

```
ServerMoveHandleClientError(ClientTimeStamp, DeltaTime, Accel, ClientLoc ...)
  ├─ 位置誤差 = |ServerLoc - ClientLoc|²
  ├─ < MAXPOSITIONERRORSQUARED (≈ 3.0cm²) → 承認（ACK のみ）
  └─ ≥ 閾値
       ├─ 通常移動: ClientAdjustPosition() RPC
       └─ RootMotion: ClientAdjustRootMotionSourcePosition() RPC
```

---

## FNetworkPredictionData_Client_Character

クライアント側のネット予測状態コンテナ:

```cpp
class FNetworkPredictionData_Client_Character : public FNetworkPredictionData_Client
{
    TArray<FSavedMovePtr> SavedMoves;      // 未確認の移動リスト
    TArray<FSavedMovePtr> FreeMoves;       // プールされた SavedMove（再利用）
    FSavedMovePtr         PendingMove;     // 次の送信待ち Move（Combine 用）
    FSavedMovePtr         LastAckedMove;   // 最後にサーバーに承認された Move

    float CurrentTimeStamp;               // クライアント時刻
};
```

---

## SimulatedProxy の補間

他プレイヤーの動き（SimulatedProxy）はサーバーから位置を受け取り、
クライアント側で補間して表示する:

```
[Server → Client] リプリケートされた Transform + Velocity
  └─ SmoothClientPosition()             [CMC.cpp]
       ├─ SmoothClientPosition_Interpolate()   — 線形補間
       └─ SmoothClientPosition_UpdateVisuals() — メッシュ位置だけ移動（コリジョンは即時）
```

補間パラメータ:
```cpp
// NetworkSmoothingMode
// Linear:   線形補間（デフォルト）
// Exponential: 指数補間（よりスムース）
// Replay:   リプレイ時専用
UPROPERTY(EditDefaultsOnly, Category=Smoothing)
ENetworkSmoothingMode NetworkSmoothingMode = ENetworkSmoothingMode::Linear;
```

---

## カスタム CMC でのネット予測拡張

```cpp
// 1. SavedMove を継承してカスタム状態を追加
// 2. FNetworkPredictionData_Client_Character を継承してプールを拡張
// 3. GetPredictionData_Client() をオーバーライドして返す

class UMyCharacterMovementComponent : public UCharacterMovementComponent
{
    virtual FNetworkPredictionData_Client* GetPredictionData_Client() const override
    {
        if (!ClientPredictionData)
        {
            UMyCharacterMovementComponent* MutableThis = const_cast<UMyCharacterMovementComponent*>(this);
            MutableThis->ClientPredictionData = new FMyNetworkPredictionData_Client_Character(*this);
        }
        return ClientPredictionData;
    }
};
```

---

## デバッグ CVar

| CVar | 説明 |
|------|------|
| `p.NetShowCorrections 1` | 位置補正を可視化 |
| `p.NetCorrectionLifetime 4` | 補正ラインの表示時間 |
| `log LogNetPlayerMovement Verbose` | ServerMove ログ |
| `p.SavedMoveVisibility 1` | SavedMove の可視化 |

---

## 関連ドキュメント

- [[a_movement_modes]] — 各物理モードの計算
- [[c_root_motion]] — RootMotion 付き移動のネット同期
- [[Reference/ref_cmc_api]] — CMC API
