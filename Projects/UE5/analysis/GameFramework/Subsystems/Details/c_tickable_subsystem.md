# Tickable Subsystem — FTickableGameObject / UTickableWorldSubsystem

- 上位: [[Subsystems/01_overview]]
- 関連: [[a_subsystem_types]] | [[b_subsystem_lifecycle]]
- ソース: `Engine/Source/Runtime/Engine/Public/Subsystems/WorldSubsystem.h`, `Engine/Source/Runtime/Engine/Public/Tickable.h`

---

## 概要

Subsystem はデフォルトでは Tick を持たない。Tick が必要な場合は 2 つの方法がある:

| 方法 | 対象 | 特徴 |
|------|------|------|
| `UTickableWorldSubsystem` を継承 | WorldSubsystem のみ | UE 推奨・FTickableGameObject が組み込み済み |
| `FTickableGameObject` を多重継承 | どの Subsystem でも | 手動で Register/Unregister が必要 |

---

## UTickableWorldSubsystem

```cpp
UCLASS(Abstract)
class UTickableWorldSubsystem : public UWorldSubsystem, public FTickableGameObject
{
    // FTickableGameObject の実装が組み込み済み
    virtual ETickableTickType GetTickableTickType() const override;
    virtual bool IsAllowedToTick() const override final; // Initialize 後に true になる
    virtual void Tick(float DeltaTime) override;         // 派生でオーバーライド
    virtual TStatId GetStatId() const override PURE_VIRTUAL(...);

    // USubsystem の実装（Tick の有効化/無効化と連動）
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;
    virtual void BeginDestroy() override;

    bool IsInitialized() const { return bInitialized; }
private:
    bool bInitialized = false;
};
```

---

## 実装例 — UTickableWorldSubsystem

```cpp
// MyTimerSubsystem.h
UCLASS()
class UMyTimerSubsystem : public UTickableWorldSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;
    virtual void Tick(float DeltaTime) override;

    // PURE_VIRTUAL なので必ず実装する
    virtual TStatId GetStatId() const override
    {
        RETURN_QUICK_DECLARE_CYCLE_STAT(UMyTimerSubsystem, STATGROUP_Tickables);
    }

    void RegisterCountdown(float Duration, FSimpleDelegate OnComplete);

private:
    struct FCountdown
    {
        float Remaining;
        FSimpleDelegate Callback;
    };
    TArray<FCountdown> ActiveCountdowns;
};

// MyTimerSubsystem.cpp
void UMyTimerSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);  // 必須: bInitialized = true になる
}

void UMyTimerSubsystem::Deinitialize()
{
    ActiveCountdowns.Empty();
    Super::Deinitialize();          // 必須: bInitialized = false になる
}

void UMyTimerSubsystem::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);         // 通常 UWorldSubsystem は何もしないが呼ぶ

    for (int32 i = ActiveCountdowns.Num() - 1; i >= 0; --i)
    {
        ActiveCountdowns[i].Remaining -= DeltaTime;
        if (ActiveCountdowns[i].Remaining <= 0.f)
        {
            ActiveCountdowns[i].Callback.ExecuteIfBound();
            ActiveCountdowns.RemoveAt(i);
        }
    }
}
```

---

## FTickableGameObject — 直接実装

`UGameInstanceSubsystem` や `ULocalPlayerSubsystem` に Tick をつけたい場合:

```cpp
// FTickableGameObject をミックスイン
UCLASS()
class UMyGameInstanceSubsystem
    : public UGameInstanceSubsystem
    , public FTickableGameObject
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // FTickableGameObject
    virtual void Tick(float DeltaTime) override;
    virtual TStatId GetStatId() const override
    {
        RETURN_QUICK_DECLARE_CYCLE_STAT(UMyGameInstanceSubsystem, STATGROUP_Tickables);
    }
    virtual bool IsTickable() const override { return bShouldTick; }
    virtual bool IsTickableInEditor() const override { return false; }

private:
    bool bShouldTick = false;
};

// Initialize/Deinitialize での Register/Unregister は不要
// FTickableGameObject はコンストラクタで自動登録、デストラクタで自動解除
```

---

## ETickableTickType

Tick の有効化タイミングを制御:

```cpp
enum class ETickableTickType : uint8
{
    Never,      // Tick しない
    Conditional, // IsTickable() が true の時のみ（デフォルト）
    Always,     // IsTickable() に関わらず常に Tick
};

// UTickableWorldSubsystem のデフォルト
// → IsAllowedToTick() = (bInitialized == true) で制御
```

---

## GetStatId の実装（必須）

`PURE_VIRTUAL` のため実装を忘れるとリンクエラーが出る:

```cpp
// マクロで簡単に宣言
virtual TStatId GetStatId() const override
{
    RETURN_QUICK_DECLARE_CYCLE_STAT(UMySubsystem, STATGROUP_Tickables);
}
```

`STATGROUP_Tickables` に追加されることでプロファイラに表示される:

```
Unreal Insights → CPU Tracks → Tickables → UMySubsystem
stat Tickables          // コンソールコマンドで表示
```

---

## Tick 頻度の制御

FTickableGameObject には TickInterval 概念がない（常にエンジン Tick ごとに呼ばれる）。頻度を下げるには自前で間引く:

```cpp
void UMySubsystem::Tick(float DeltaTime)
{
    AccumulatedTime += DeltaTime;
    if (AccumulatedTime < UpdateInterval)
        return;

    AccumulatedTime = 0.f;
    DoHeavyUpdate();
}
```

または `FTimerManager` を使う:

```cpp
void UMySubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    GetWorld()->GetTimerManager().SetTimer(
        TimerHandle, this, &UMySubsystem::OnTimer, 1.0f, /*bLoop=*/ true
    );
}

void UMySubsystem::Deinitialize()
{
    if (GetWorld())
        GetWorld()->GetTimerManager().ClearTimer(TimerHandle);
    Super::Deinitialize();
}
```

---

## IsTickableInEditor / IsTickableWhenPaused

```cpp
// エディタでも Tick する（ツール系 Subsystem に有用）
virtual bool IsTickableInEditor() const override { return true; }

// ゲームポーズ中も Tick する（タイムライン再生等）
virtual bool IsTickableWhenPaused() const override { return true; }
```

---

## 関連ドキュメント

- [[a_subsystem_types]] — Subsystem の種類と生存期間
- [[b_subsystem_lifecycle]] — Initialize / Deinitialize の詳細
- [[Reference/ref_subsystem_api]] — USubsystem / FTickableGameObject API
