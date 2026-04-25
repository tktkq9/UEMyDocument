# Subsystem の種類 — GameInstance / World / LocalPlayer / Engine

- 上位: [[Subsystems/01_overview]]
- 関連: [[b_subsystem_lifecycle]] | [[c_tickable_subsystem]]
- ソース: `Engine/Source/Runtime/Engine/Public/Subsystems/Subsystem.h`, `GameInstanceSubsystem.h`, `WorldSubsystem.h`, `LocalPlayerSubsystem.h`, `EngineSubsystem.h`

---

## 概要

UE5 のサブシステムは「特定オブジェクトの生存期間を共有し、自動でインスタンス化・破棄される `UObject` ベースのシングルトン」。`GetSubsystem<T>()` で取得できる型安全な代替手段として `GameInstance` 変数や `static` を置き換えることができる。

---

## 基底クラス

```cpp
UCLASS(Abstract)
class USubsystem : public UObject
{
    // ─── オーバーライドポイント ──────────────────────────────
    virtual bool ShouldCreateSubsystem(UObject* Outer) const { return true; }
    virtual void Initialize(FSubsystemCollectionBase& Collection) {}
    virtual void Deinitialize() {}
};
```

---

## 種類と生存期間

| クラス | 生存期間 | Outer | 取得方法 |
|--------|---------|-------|---------|
| `UEngineSubsystem` | GEngine 生存中（アプリ起動〜終了） | GEngine | `GEngine->GetEngineSubsystem<T>()` |
| `UGameInstanceSubsystem` | GameInstance と同じ（セッション単位） | UGameInstance | `UGameInstance->GetSubsystem<T>()` |
| `UWorldSubsystem` | UWorld と同じ（レベルロード〜アンロード） | UWorld | `World->GetSubsystem<T>()` |
| `ULocalPlayerSubsystem` | ULocalPlayer と同じ（接続〜切断） | ULocalPlayer | `LocalPlayer->GetSubsystem<T>()` |
| `UEditorSubsystem` | エディタ起動中（ランタイム不可） | UEditorEngine | `GEditor->GetEditorSubsystem<T>()` |

---

## UGameInstanceSubsystem

```cpp
UCLASS(Abstract, Within = GameInstance)
class UGameInstanceSubsystem : public USubsystem
{
    ENGINE_API UGameInstance* GetGameInstance() const;
};
```

**用途**: セーブシステム・マッチメイキング・グローバル設定など、レベルまたぎで状態を保持したいもの。

```cpp
// 定義
UCLASS()
class UMySaveSubsystem : public UGameInstanceSubsystem
{
    GENERATED_BODY()
public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    void SaveGame();
    void LoadGame();

private:
    UPROPERTY()
    TObjectPtr<UMySaveGame> CurrentSaveGame;
};

// 取得（どこからでも）
UGameInstance* GI = GetWorld()->GetGameInstance();
UMySaveSubsystem* SaveSys = GI->GetSubsystem<UMySaveSubsystem>();
SaveSys->SaveGame();
```

---

## UWorldSubsystem

```cpp
UCLASS(Abstract)
class UWorldSubsystem : public USubsystem
{
    ENGINE_API virtual UWorld* GetWorld() const override final;

    // ─── ライフサイクル追加フック ────────────────────────────
    virtual void PostInitialize();             // 全 WorldSubsystem 初期化後
    virtual void PreDeinitialize() {}
    virtual void OnWorldBeginPlay(UWorld& InWorld);
    virtual void OnWorldEndPlay(UWorld& InWorld) {}
    virtual void OnWorldComponentsUpdated(UWorld& World) {}

protected:
    // 特定 WorldType のみ作成するフィルタリング
    virtual bool DoesSupportWorldType(const EWorldType::Type WorldType) const;
};
```

**用途**: ゲームプレイ中の一時的なグローバル状態（敵グループ管理・ウェーブシステム等）。

```cpp
// DoesSupportWorldType でエディタやプレビューでは作らない
bool UMyWorldSubsystem::DoesSupportWorldType(EWorldType::Type WorldType) const
{
    return WorldType == EWorldType::Game || WorldType == EWorldType::PIE;
}
```

---

## ULocalPlayerSubsystem

```cpp
UCLASS(Abstract, Within = LocalPlayer)
class ULocalPlayerSubsystem : public USubsystem
{
    template<typename T = ULocalPlayer>
    T* GetLocalPlayer() const;

    // PlayerController が変わった時のコールバック
    virtual void PlayerControllerChanged(APlayerController* NewPlayerController);
};
```

**用途**: プレイヤー固有の設定（コントローラー感度・UI 状態）、スプリットスクリーン対応。

```cpp
// 取得
ULocalPlayer* LP = GetWorld()->GetFirstLocalPlayerFromController();
UMyUISubsystem* UISys = LP->GetSubsystem<UMyUISubsystem>();

// PlayerController 変更時の通知
void UMyUISubsystem::PlayerControllerChanged(APlayerController* NewPC)
{
    Super::PlayerControllerChanged(NewPC);
    // 新しい PC に HUD を設定し直す
}
```

---

## UEngineSubsystem

```cpp
UCLASS(Abstract)
class UEngineSubsystem : public UDynamicSubsystem
{
    // GEngine が作成・管理
};
```

**用途**: ゲームに依存しないエンジンレベルのサービス（テレメトリ送信、クラッシュレポーター連携等）。プラグインのエントリポイントとしても使われる。

```cpp
// 取得
UMyEngineSubsystem* EngSys = GEngine->GetEngineSubsystem<UMyEngineSubsystem>();
```

---

## GetSubsystem のヘルパー関数

```cpp
// World から各種 Subsystem を取得するショートカット
UGameInstance* GI = World->GetGameInstance();
UMySubsystem* Sys = UGameInstance::GetSubsystem<UMySubsystem>(GI); // null-safe

// ブループリントからは SubsystemBlueprintLibrary を使う
// Get Game Instance Subsystem ノード
// Get World Subsystem ノード
// Get Local Player Subsystem ノード
```

---

## 複数実装（インターフェース）

同じインターフェースを実装する複数の Subsystem を取得することもできる:

```cpp
// インターフェース
class IMySystemInterface { /* 純粋仮想関数 */ };

// 実装 A, B を登録
class UMySystemA : public UGameInstanceSubsystem, public IMySystemInterface {};
class UMySystemB : public UGameInstanceSubsystem, public IMySystemInterface {};

// 全実装を取得
const TArray<UMySystemInterface*>& Systems =
    GameInstance->GetSubsystemArray<UMySystemInterface>();
```

---

## 関連ドキュメント

- [[b_subsystem_lifecycle]] — Initialize / Deinitialize / ShouldCreateSubsystem
- [[c_tickable_subsystem]] — FTickableGameObject / UTickableWorldSubsystem
- [[Reference/ref_subsystem_api]] — USubsystem API
