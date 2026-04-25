# USubsystem / FSubsystemCollectionBase API リファレンス

- 上位: [[Subsystems/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Public/Subsystems/Subsystem.h`, `SubsystemCollection.h`, `WorldSubsystem.h`

---

## USubsystem — 基底クラス

```cpp
UCLASS(Abstract)
class USubsystem : public UObject
{
    // ─── ライフサイクル ────────────────────────────────────────
    // CDO に対して呼ばれる。Outer はまだ初期化されていない。
    virtual bool ShouldCreateSubsystem(UObject* Outer) const { return true; }

    // インスタンスに対して呼ばれる
    virtual void Initialize(FSubsystemCollectionBase& Collection) {}
    virtual void Deinitialize() {}
};
```

---

## 各 Subsystem 型と取得方法

### UGameInstanceSubsystem

```cpp
UCLASS(Abstract, Within = GameInstance)
class UGameInstanceSubsystem : public USubsystem
{
    UGameInstance* GetGameInstance() const;
};

// 取得
UGameInstance* GI = World->GetGameInstance();
UMySubsystem* Sys = GI->GetSubsystem<UMySubsystem>();
UMySubsystem* Sys = UGameInstance::GetSubsystem<UMySubsystem>(GI); // null-safe
```

### UWorldSubsystem

```cpp
UCLASS(Abstract)
class UWorldSubsystem : public USubsystem
{
    virtual UWorld* GetWorld() const override final;

    // 追加ライフサイクル
    virtual void PostInitialize();
    virtual void PreDeinitialize() {}
    virtual void OnWorldBeginPlay(UWorld& InWorld);
    virtual void OnWorldEndPlay(UWorld& InWorld) {}
    virtual void OnWorldComponentsUpdated(UWorld& World) {}

protected:
    virtual bool DoesSupportWorldType(const EWorldType::Type WorldType) const;
};

// 取得
UMyWorldSub* Sys = World->GetSubsystem<UMyWorldSub>();
```

### UTickableWorldSubsystem

```cpp
UCLASS(Abstract)
class UTickableWorldSubsystem : public UWorldSubsystem, public FTickableGameObject
{
    // Tick は Initialize 後〜Deinitialize まで有効
    virtual void Tick(float DeltaTime) override;
    virtual TStatId GetStatId() const override PURE_VIRTUAL(...); // 必須実装

    bool IsInitialized() const;
};
```

### ULocalPlayerSubsystem

```cpp
UCLASS(Abstract, Within = LocalPlayer)
class ULocalPlayerSubsystem : public USubsystem
{
    template<typename T = ULocalPlayer>
    T* GetLocalPlayer() const;

    virtual void PlayerControllerChanged(APlayerController* NewPlayerController);
};

// 取得
ULocalPlayer* LP = World->GetFirstLocalPlayerFromController();
UMyLPSub* Sys = LP->GetSubsystem<UMyLPSub>();
```

### UEngineSubsystem

```cpp
UCLASS(Abstract)
class UEngineSubsystem : public UDynamicSubsystem {};

// 取得
UMyEngineSub* Sys = GEngine->GetEngineSubsystem<UMyEngineSub>();
```

---

## FSubsystemCollectionBase — コレクション管理

```cpp
class FSubsystemCollectionBase
{
    void Initialize(UObject* NewOuter);    // Outer の初期化時に自動呼び出し
    void Deinitialize();                   // Outer 破棄時に自動呼び出し
    bool IsInitialized() const;

    // Initialize() 内でのみ呼ぶ（依存関係の解決）
    template<class T>
    T* InitializeDependency();
    USubsystem* InitializeDependency(TSubclassOf<USubsystem> SubsystemClass);

    // 取得
    USubsystem* GetSubsystemInternal(UClass* SubsystemClass) const;
    TArray<USubsystem*> GetSubsystemArrayCopy(UClass* SubsystemClass) const;

    // 動的登録（プラグイン用）
    static void ActivateExternalSubsystem(UClass* SubsystemClass);
    static void DeactivateExternalSubsystem(UClass* SubsystemClass);
};
```

---

## FTickableGameObject — Tick の実装インターフェース

```cpp
class FTickableGameObject
{
    // 必須オーバーライド
    virtual void Tick(float DeltaTime) = 0;
    virtual TStatId GetStatId() const = 0;   // PURE_VIRTUAL — 必ず実装

    // 任意オーバーライド
    virtual ETickableTickType GetTickableTickType() const { return ETickableTickType::Conditional; }
    virtual bool IsTickable() const { return true; }
    virtual bool IsTickableInEditor() const { return false; }
    virtual bool IsTickableWhenPaused() const { return false; }
    virtual UWorld* GetTickableGameObjectWorld() const { return nullptr; }
};

// GetStatId の実装パターン
virtual TStatId GetStatId() const override
{
    RETURN_QUICK_DECLARE_CYCLE_STAT(UMySubsystem, STATGROUP_Tickables);
}
```

---

## Blueprint ヘルパー（USubsystemBlueprintLibrary）

```cpp
// Blueprint から各 Subsystem にアクセスするノード群
class USubsystemBlueprintLibrary : public UBlueprintFunctionLibrary
{
    static UGameInstanceSubsystem* GetGameInstanceSubsystem(
        UObject* ContextObject, TSubclassOf<UGameInstanceSubsystem> Class);

    static UWorldSubsystem* GetWorldSubsystem(
        UObject* ContextObject, TSubclassOf<UWorldSubsystem> Class);

    static ULocalPlayerSubsystem* GetLocalPlayerSubsystem(
        UObject* ContextObject, TSubclassOf<ULocalPlayerSubsystem> Class);

    static UEngineSubsystem* GetEngineSubsystem(
        TSubclassOf<UEngineSubsystem> Class);
};
// Blueprint ノード名: Get Game Instance Subsystem / Get World Subsystem 等
```

---

## 複数実装（インターフェース）の取得

```cpp
// GetSubsystem は最初の 1 件を返す
// 複数実装を全取得する場合
const TArray<UMyInterface*>& All = GI->GetSubsystemArray<UMyInterface>();
```

---

## EWorldType（DoesSupportWorldType で使用）

```cpp
namespace EWorldType
{
    enum Type
    {
        None,       // 未初期化
        Game,       // ゲーム実行中
        Editor,     // エディタ
        PIE,        // Play In Editor
        EditorPreview, // エディタプレビュービューポート
        GamePreview,
        GameRPC,
        Inactive,
    };
}
```

---

## 関連ドキュメント

- [[Details/a_subsystem_types]] — 各 Subsystem の生存期間
- [[Details/b_subsystem_lifecycle]] — Initialize / ShouldCreateSubsystem
- [[Details/c_tickable_subsystem]] — Tick 実装の詳細
