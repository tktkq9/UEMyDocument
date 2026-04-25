# Subsystem ライフサイクル — Initialize / Deinitialize / ShouldCreateSubsystem / 依存関係

- 上位: [[Subsystems/01_overview]]
- 関連: [[a_subsystem_types]] | [[c_tickable_subsystem]]
- ソース: `Engine/Source/Runtime/Engine/Public/Subsystems/SubsystemCollection.h`, `Subsystem.h`, `WorldSubsystem.h`

---

## 概要

Subsystem のインスタンス生成・初期化・破棄は `FSubsystemCollection<T>` が管理する。`Outer` オブジェクト（GameInstance 等）が初期化されると `FSubsystemCollectionBase::Initialize()` が走り、登録済みの全 Subsystem クラスがインスタンス化される。

---

## ライフサイクル全体

```
[Outer 生成時]
FSubsystemCollectionBase::Initialize(Outer)   [SubsystemCollection.h]
  ├─ GetDerivedClasses(BaseType) — 全サブクラスを発見
  ├─ for each SubsystemClass:
  │    ├─ ShouldCreateSubsystem(Outer) → false なら skip
  │    └─ AddAndInitializeSubsystem(Class)
  │         ├─ NewObject<USubsystem>(Outer, Class)
  │         └─ Subsystem->Initialize(Collection)
  └─ (UWorldSubsystem のみ) PostInitialize() — 全 WS 初期化後に呼ばれる

[通常ゲームプレイ中]
  各 Subsystem は自分のロジックを Tick や Event で実行

[Outer 破棄時]
FSubsystemCollectionBase::Deinitialize()
  └─ for each Subsystem in 逆順:
       └─ Subsystem->Deinitialize()
```

---

## Initialize — 初期化

```cpp
virtual void Initialize(FSubsystemCollectionBase& Collection) override
{
    // Collection = 同じ Outer の SubsystemCollection
    // ここで依存する他 Subsystem を要求できる
    UMyDependency* Dep = Collection.InitializeDependency<UMyDependency>();

    // デリゲートをバインド
    if (UGameInstance* GI = GetGameInstance())
    {
        GI->OnStartDelegate.AddUObject(this, &UMySubsystem::OnGameStart);
    }
}
```

`Initialize` は **CDO ではなくインスタンスに対して**呼ばれる。`GetOuter()` は有効な Outer（GameInstance 等）を返す。

---

## Deinitialize — 終了

```cpp
virtual void Deinitialize() override
{
    // バインドしたデリゲートを必ず解除
    if (UGameInstance* GI = GetGameInstance())
    {
        GI->OnStartDelegate.RemoveAll(this);
    }

    // 保持リソースのクリア
    CachedData.Empty();
}
```

`Deinitialize` の呼び出し順は `Initialize` の逆順（後から初期化されたものが先に終了）。

---

## ShouldCreateSubsystem — 条件付き生成

```cpp
virtual bool ShouldCreateSubsystem(UObject* Outer) const override
{
    // NOTE: CDO（クラスデフォルトオブジェクト）に対して呼ばれる。
    // Outer はまだ初期化されていない可能性があるため、
    // Outer のプロパティには安易にアクセスしない。

    // ゲームの World でのみ作成（エディタ・プレビュー除外）
    if (const UWorld* World = Cast<UWorld>(Outer))
    {
        if (World->WorldType != EWorldType::Game &&
            World->WorldType != EWorldType::PIE)
        {
            return false;
        }
    }

    // プロジェクト設定でフラグが立っている場合のみ
    // （GetDefault<> は CDO から安全にアクセスできる）
    return GetDefault<UMyGameSettings>()->bEnableMySubsystem;
}
```

> `ShouldCreateSubsystem` が `false` を返した場合、`GetSubsystem<T>()` は `nullptr` を返す。呼び出し側で必ず null チェックを行う。

---

## InitializeDependency — 依存関係の宣言

同じコレクション内の別 Subsystem を先に初期化させる:

```cpp
void UMySubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    // UMyDependency が必ず先に Initialize される
    UMyDependency* Dep = Collection.InitializeDependency<UMyDependency>();
    check(Dep);   // 失敗時はログと nullptr

    // 以降は Dep->GetSomeData() などが安全に使える
    CachedValue = Dep->GetInitialValue();
}
```

> `InitializeDependency` は同じ `FSubsystemCollection` 内のみ有効（GameInstance 系から World 系の依存は解決できない）。

---

## UWorldSubsystem の追加フック

`UWorldSubsystem` は `USubsystem` の Initialize / Deinitialize に加えて、レベル・ゲームプレイのタイミングに合わせたフックを持つ:

```cpp
class UMyWorldSubsystem : public UWorldSubsystem
{
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void PostInitialize() override;          // 全 WorldSubsystem 初期化後
    virtual void PreDeinitialize() override;         // Deinitialize 前
    virtual void Deinitialize() override;
    virtual void OnWorldBeginPlay(UWorld& InWorld) override;  // BeginPlay 前
    virtual void OnWorldEndPlay(UWorld& InWorld) override;
    virtual void OnWorldComponentsUpdated(UWorld& World) override; // レベル更新後
};
```

フックの呼び出し順:

```
Initialize()               ← Outer(World) 生成直後
  └─ PostInitialize()      ← 全 WorldSubsystem 初期化完了後

OnWorldBeginPlay()         ← GameMode が BeginPlay に遷移する直前
  └─ [各 Actor の BeginPlay]

OnWorldEndPlay()           ← EndPlay フェーズ開始前

PreDeinitialize()          ← Deinitialize 前（クリーンアップ準備）
Deinitialize()             ← World 破棄時
```

---

## ULocalPlayerSubsystem の追加フック

```cpp
class UMyLocalPlayerSubsystem : public ULocalPlayerSubsystem
{
    // PlayerController が変わるたびに呼ばれる（スプリットスクリーン等）
    virtual void PlayerControllerChanged(APlayerController* NewPlayerController) override
    {
        Super::PlayerControllerChanged(NewPlayerController);
        // HUD の再設定、入力バインドのリセット等
    }
};
```

---

## DynamicSubsystem — プラグインの動的登録

プラグインの有効化・無効化に連動して Subsystem を追加・削除できる:

```cpp
// プラグイン有効化時
FSubsystemCollectionBase::ActivateExternalSubsystem(UMyPluginSubsystem::StaticClass());

// プラグイン無効化時
FSubsystemCollectionBase::DeactivateExternalSubsystem(UMyPluginSubsystem::StaticClass());
```

---

## よくあるミス

| ミス | 正しい対処 |
|------|-----------|
| `ShouldCreateSubsystem` で `GetOuter()` にキャスト | CDO なので Outer は nullptr。`Outer` 引数を使う |
| `Initialize` 外で `InitializeDependency` 呼び出し | `Initialize(Collection)` の引数 `Collection` からのみ呼ぶ |
| Deinitialize でデリゲートを外し忘れ | `RemoveAll(this)` パターンで確実に解除 |
| `GetSubsystem` の null チェックなし | `ShouldCreateSubsystem` が false ならば null |

---

## 関連ドキュメント

- [[a_subsystem_types]] — 各 Subsystem の種類と生存期間
- [[c_tickable_subsystem]] — Tick が必要な Subsystem
- [[Reference/ref_subsystem_api]] — USubsystem / FSubsystemCollectionBase API
