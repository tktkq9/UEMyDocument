# ティック（FTickFunction / TickGroup）

- 上位: [[ActorComponent/01_overview]]
- 関連: [[a_actor_lifecycle]] | [[b_component_model]]
- ソース: `Engine/Source/Runtime/Engine/Classes/Engine/EngineBaseTypes.h`, `Engine/Source/Runtime/Engine/Private/TickTaskManager.cpp`

---

## 概要

UE5 のティックは **`FTickFunction` ベースのスケジューラ** で管理される。Actor と Component はそれぞれ独自の `FActorTickFunction` / `FActorComponentTickFunction` を持ち、`TickGroup` と Prerequisites（前提条件）によって実行順を制御できる。

---

## TickGroup（実行グループ）

| グループ | 値 | 実行タイミング |
|---------|---|--------------|
| `TG_PrePhysics` | 0 | 物理シミュレーション開始前（デフォルト） |
| `TG_DuringPhysics` | 1 | 物理シミュレーションと並列（非同期） |
| `TG_PostPhysics` | 2 | 物理シミュレーション完了後 |
| `TG_PostUpdateWork` | 3 | 全物理更新後（カメラ更新等） |
| `TG_LastDemotable` | 4 | Prerequisites が満たせない場合の受け皿 |

```
フレーム開始
  │
  ├─ TG_PrePhysics  → Actor Tick / CMC 入力処理
  │
  ├─ 物理シミュレーション（Chaos / PhysX）
  │    └─ TG_DuringPhysics（並列実行）
  │
  ├─ TG_PostPhysics → 物理結果読み取り（布・物理オブジェクト）
  │
  └─ TG_PostUpdateWork → カメラ / スプリングアーム / アニメ最終更新
```

---

## Actor ティック設定

```cpp
// Actor コンストラクタで設定
AMyActor::AMyActor()
{
    // Tick を有効化（デフォルトは false）
    PrimaryActorTick.bCanEverTick = true;

    // TickGroup 指定（デフォルト: TG_PrePhysics）
    PrimaryActorTick.TickGroup = TG_PostPhysics;

    // Tick 間隔指定（0 = 毎フレーム）
    PrimaryActorTick.TickInterval = 0.1f;  // 0.1 秒ごと

    // 一時的に Tick を止めるなら
    // SetActorTickEnabled(false); // BeginPlay 以降に使用
}

virtual void Tick(float DeltaSeconds) override
{
    Super::Tick(DeltaSeconds);
    // 処理
}
```

---

## Component ティック設定

```cpp
UMyComponent::UMyComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
    PrimaryComponentTick.TickGroup = TG_PostPhysics;
    PrimaryComponentTick.bStartWithTickEnabled = true;
}

virtual void TickComponent(float DeltaTime, ELevelTick TickType,
                            FActorComponentTickFunction* ThisTickFunction) override
{
    Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
    // 処理
}
```

---

## Tick の前提条件（Prerequisites）

他の Actor / Component が Tick 完了後に自分が Tick するよう依存関係を設定:

```cpp
// BeginPlay 内で設定
void AMyActor::BeginPlay()
{
    Super::BeginPlay();

    // OtherActor の Tick 後に自分が Tick
    AddTickPrerequisiteActor(OtherActor);

    // OtherComponent の Tick 後に自分が Tick
    AddTickPrerequisiteComponent(SomeComponent);
}

// コンポーネント側での設定
void UMyComp::BeginPlay()
{
    Super::BeginPlay();

    // 特定 Component の後に TickComponent を実行
    PrimaryComponentTick.AddPrerequisite(GetOwner(),
        GetOwner()->GetCharacterMovement()->PrimaryComponentTick);
}
```

---

## FTickFunction の内部構造

```cpp
struct FTickFunction
{
    // 設定プロパティ
    bool bCanEverTick;           // Tick する可能性があるか（初期値でのみ効果あり）
    bool bStartWithTickEnabled;  // 開始時に有効か
    bool bTickEvenWhenPaused;    // ポーズ中も Tick するか
    bool bHighPriority;          // 高優先度（同じグループ内で早く実行）

    TEnumAsByte<ETickingGroup> TickGroup;     // 実行グループ（目標）
    TEnumAsByte<ETickingGroup> EndTickGroup;  // 最遅グループ（prerequisites 遅延の上限）

    float TickInterval;          // 最低 Tick 間隔（秒）。0 = 毎フレーム

    // Prerequisites
    void AddPrerequisite(UObject* TargetObject, FTickFunction& TargetTickFunction);
    void RemovePrerequisite(UObject* TargetObject, FTickFunction& TargetTickFunction);
};
```

---

## TickTaskManager の仕組み

```
FTickTaskManager::StartFrame()                          [TickTaskManager.cpp]
  └─ 全 TickFunction を TickGroup ごとにソート
       └─ Prerequisites → 実際の実行 Group が後ろにずれることがある
            (TG_LastDemotable が最後の受け皿)

  └─ TG_PrePhysics の全 TickFunction を実行（並列化あり）
       └─ 物理更新開始
  └─ TG_DuringPhysics の TickFunction を並列実行
       └─ 物理完了待ち
  └─ TG_PostPhysics の TickFunction を実行
  └─ TG_PostUpdateWork の TickFunction を実行
```

---

## TickInterval の動作

```cpp
// TickInterval = 0.1f の場合、Tick は最低 0.1 秒ごと
// ただし実際の DeltaTime は 0.1 以上になる可能性がある（フレームレート依存）

// 毎フレーム正確に実行したい場合は TickInterval = 0 (デフォルト)
// CPU 節約のために間引く場合は 0.05f ～ 0.5f 程度
```

---

## よくあるパターン

```cpp
// パターン 1: ゲーム開始後に動的に有効化
void AMyActor::StartTicking()
{
    SetActorTickEnabled(true);
}

// パターン 2: コンポーネントごとに制御
UMyComp* Comp = GetComponentByClass<UMyComp>();
Comp->SetComponentTickEnabled(false);

// パターン 3: TickGroup 変更（BeginPlay 後も可能）
PrimaryActorTick.TickGroup = TG_PostPhysics;

// パターン 4: Tick に依存する Timer の代替（低負荷）
GetWorldTimerManager().SetTimer(TimerHandle, this, &AMyActor::OnInterval, 0.1f, true);
// → Tick を使わず Timer で間引き実行
```

---

## 関連ドキュメント

- [[a_actor_lifecycle]] — BeginPlay → EndPlay の全体フロー
- [[b_component_model]] — コンポーネントの TickComponent
- [[../../Delegates/Details/c_timer]] — FTimerManager（Tick の代替）
