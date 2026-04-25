# RootMotion — AnimMontage による移動制御

- 上位: [[CharacterMovement/01_overview]]
- 関連: [[a_movement_modes]] | [[b_cmc_networking]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/RootMotionSource.h`, `CharacterMovementComponent.h`, `CharacterMovementComponent.cpp`

---

## 概要

RootMotion とは**アニメーションのルートボーンの変位をキャラクターの実際の移動量として使う仕組み**。攻撃モーションで前進・ジャンプ着地で後退など、アーティストが意図した動きをそのまま物理空間に反映できる。

UE の RootMotion は 2 系統ある:

| 種別 | 発生源 | 用途 |
|------|--------|------|
| AnimRootMotion | AnimMontage / AnimSequence の RootBone 変位 | アニメーション主導の近接攻撃・特殊モーション |
| RootMotionSource | コードで生成する FRootMotionSource | Gameplay Ability System の AbilityTask で使う（ダッシュ・ノックバック等） |

---

## AnimRootMotion のフロー

```
[アニメーション更新]
USkeletalMeshComponent::TickAnimation()
  └─ UAnimInstance::UpdateAnimation()
       └─ RootBone 変位を FRootMotionMovementParams に蓄積

[CMC 移動フレーム]
UCharacterMovementComponent::PerformMovement(DeltaTime)
  ├─ ConsumeRootMotion()                 ← AnimInstance から変位を取得
  │    └─ FRootMotionMovementParams に格納（RootMotionParams）
  ├─ HasAnimRootMotion() == true なら:
  │    ├─ CalcAnimRootMotionVelocity()   — 変位 / DeltaTime → 速度へ変換
  │    ├─ ApplyRootMotionToVelocity()    — RootMotion が Velocity を上書き
  │    └─ PhysWalking / PhysFalling 内で通常ではなく RootMotion 速度を使用
  └─ RootMotionParams をリセット
```

---

## AnimMontage で RootMotion を有効にする

```cpp
// UAnimMontage.h 設定
// ContentBrowser でモンタージュの RootMotionRootLock や EnableRootMotionTranslation にチェック

// C++ から再生
void AMyCharacter::PerformAttack()
{
    if (UAnimInstance* AnimInst = GetMesh()->GetAnimInstance())
    {
        AnimInst->Montage_Play(AttackMontage, /*PlayRate=*/ 1.0f);
    }
}

// Blueprint: Play Montage ノードで同様
```

---

## RootMotionTranslationScale

アニメーションの移動量をスケールする:

```cpp
// AnimMontage 再生中に移動量を縮小（例: スロー攻撃）
GetCharacterMovement()->SetRootMotionTranslationScaleFactor(0.5f);

// スケールを取得
float Scale = GetCharacterMovement()->GetRootMotionTranslationScaleFactor();
```

---

## FRootMotionSource — コード起動型 RootMotion

`GameplayAbility` の `AbilityTask_ApplyRootMotion*` で使われるが、直接 CMC に追加もできる:

```cpp
// 利用可能な組み込み RootMotionSource
FRootMotionSource_ConstantForce   // 一定方向に力を加える（持続ノックバック等）
FRootMotionSource_RadialForce     // 爆発中心からの放射力
FRootMotionSource_MoveToForce     // 特定座標まで移動（ワープ系アビリティ）
FRootMotionSource_JumpForce       // カスタムジャンプ曲線
```

使い方:

```cpp
// 一定方向への力を加える（例: ノックバック）
TSharedPtr<FRootMotionSource_ConstantForce> KnockbackSource
    = MakeShared<FRootMotionSource_ConstantForce>();
KnockbackSource->InstanceName    = FName("Knockback");
KnockbackSource->AccumulateMode  = ERootMotionAccumulateMode::Additive;
KnockbackSource->Priority        = 5;
KnockbackSource->Force           = FVector(-500.f, 0.f, 200.f);
KnockbackSource->Duration        = 0.5f;

uint16 RootMotionSourceID = GetCharacterMovement()->ApplyRootMotionSource(KnockbackSource);

// 停止
GetCharacterMovement()->RemoveRootMotionSourceByID(RootMotionSourceID);
```

---

## ERootMotionAccumulateMode

複数の RootMotionSource が同時にある場合の合成方法:

```cpp
enum class ERootMotionAccumulateMode : uint8
{
    Override,  // 最も高い Priority の Source だけ使う
    Additive,  // 全 Source の力を加算
};
```

---

## ERootMotionFinishVelocityMode

RootMotionSource の終了時の速度処理:

```cpp
enum class ERootMotionFinishVelocityMode : uint8
{
    MaintainLastRootMotionVelocity,  // 最後の RootMotion 速度を維持（デフォルト）
    SetVelocity,                     // 指定速度にセット
    ClampVelocity,                   // 指定速度以下にクランプ
};
```

---

## ネット同期

AnimRootMotion はモンタージュ位置（MontageTrackPosition）でサーバーと同期:

```
[Server → Client]
ClientAdjustRootMotionSourcePosition() RPC
  ├─ ServerRootMotion       — RootMotionSourceGroup の状態
  ├─ ServerMontageTrackPosition — Montage の再生位置
  └─ ServerLoc              — サーバー位置
```

サーバーが再生するモンタージュと同じアセットを使う限り、クライアントは再生位置を同期するだけでよい。

---

## カスタム RootMotionSource の作成

```cpp
struct FRootMotionSource_MyDash : public FRootMotionSource
{
    FVector DashDirection;
    float   DashSpeed;

    virtual FRootMotionSource* Clone() const override
    {
        return new FRootMotionSource_MyDash(*this);
    }

    virtual bool Matches(const FRootMotionSource* Other) const override
    {
        return FRootMotionSource::Matches(Other); // InstanceName で照合
    }

    virtual void PrepareRootMotion(float SimulationTime, float MovementTickTime,
                                   const ACharacter& Character,
                                   const UCharacterMovementComponent& MoveComponent) override
    {
        // DeltaTime ぶんの移動量を RootMotionParams に書く
        RootMotionParams.Set(FTransform(DashDirection * DashSpeed * SimulationTime));
        SetTime(GetTime() + SimulationTime);
    }

    ROOTMOTION_SOURCE_BODY()  // 型情報マクロ
};
```

---

## コード実行フロー

### AnimRootMotion 適用フロー

```
USkeletalMeshComponent::TickAnimation()              [SkeletalMeshComponent.cpp]
  └─ UAnimInstance::UpdateAnimation()
       └─ RootBone 変位 → FRootMotionMovementParams に蓄積

CMC::PerformMovement(DeltaTime)                      [CMC.cpp]
  ├─ ConsumeRootMotion()                             ← AnimInstance から変位取得
  │    └─ RootMotionParams.Set(RootMotionTransform)
  ├─ HasAnimRootMotion() == true:
  │    ├─ CalcAnimRootMotionVelocity()               ← 変位 / DeltaTime → 速度
  │    └─ ApplyRootMotionToVelocity()               ← Velocity を RootMotion で上書き
  └─ PhysWalking / PhysFalling で RootMotion 速度を使って移動
```

### FRootMotionSource 適用フロー

```
CMC::ApplyRootMotionSource(SourcePtr)
  └─ CurrentRootMotion.PendingAddRootMotionSources.Add(Source)

CMC::PerformMovement()
  └─ CurrentRootMotion.PrepareRootMotion(DeltaTime, Character, CMC)
       └─ for each Source: Source->PrepareRootMotion()  ← 変位計算
  └─ ApplyRootMotionToVelocity()                    ← 合成して Velocity に適用
```

### 関与クラス・関数

| クラス | 関数 | 役割 |
|--------|------|------|
| `UAnimInstance` | `UpdateAnimation()` | RootBone 変位の計算 |
| `UCharacterMovementComponent` | `ConsumeRootMotion()` | AnimInstance から変位を取得 |
| `UCharacterMovementComponent` | `ApplyRootMotionToVelocity()` | RootMotion を速度に変換 |
| `FRootMotionSource` | `PrepareRootMotion()` | コード起動 RootMotion の変位計算 |
| `FRootMotionSourceGroup` | `PrepareRootMotion()` | 複数 Source の合成 |

---

## 関連ドキュメント

- [[a_movement_modes]] — PhysWalking / PhysFalling と RootMotion の相互作用
- [[b_cmc_networking]] — RootMotion のネット同期
- [[Reference/ref_cmc_api]] — CMC / FRootMotionSource API
