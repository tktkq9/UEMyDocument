# Actor スポーン（SpawnActorDeferred / FinishSpawning）

- 上位: [[ActorComponent/01_overview]]
- 関連: [[a_actor_lifecycle]] | [[b_component_model]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/Actor.h`, `Engine/Source/Runtime/Engine/Private/Actor.cpp`, `Engine/Source/Runtime/Engine/Private/LevelActor.cpp`

---

## 概要

Actor のスポーンには **即時スポーン**（`SpawnActor`）と **遅延スポーン**（`SpawnActorDeferred` + `FinishSpawning`）の 2 つのパターンがある。遅延スポーンは コンストラクション スクリプト実行前にプロパティを設定したい場合に使う。

---

## SpawnActor（即時スポーン）

```cpp
// C++ 基本形
FActorSpawnParameters Params;
Params.Owner     = this;
Params.Instigator = GetInstigator();
Params.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;

AMyActor* NewActor = GetWorld()->SpawnActor<AMyActor>(
    AMyActor::StaticClass(),
    SpawnTransform,
    Params
);

// テンプレート（SpawnParameters の一部を省略可）
AMyActor* Actor2 = GetWorld()->SpawnActor<AMyActor>(Location, Rotation);
```

### FActorSpawnParameters

| プロパティ | 型 | 説明 |
|-----------|---|------|
| `Owner` | `AActor*` | 所有 Actor |
| `Instigator` | `APawn*` | ダメージ等の「起因者」 |
| `Template` | `AActor*` | テンプレート Actor（CDO の代わりにコピー元） |
| `SpawnCollisionHandlingOverride` | `ESpawnActorCollisionHandlingMethod` | コリジョン衝突時の対応 |
| `bNoFail` | `bool` | `true` にするとスポーン失敗でも null を返さない |
| `bDeferConstruction` | `bool` | Construction Script 実行を遅延 |
| `OverrideLevel` | `ULevel*` | スポーン先レベル |

---

## SpawnActorDeferred（遅延スポーン）

Construction Script 実行前にプロパティを設定したい場合:

```cpp
// 1. DeferConstruction=true でスポーン（Construction Script は未実行）
AMyActor* Actor = GetWorld()->SpawnActorDeferred<AMyActor>(
    AMyActor::StaticClass(),
    SpawnTransform,
    Owner,
    Instigator,
    ESpawnActorCollisionHandlingMethod::AlwaysSpawn
);

// 2. ここでプロパティ設定（Construction Script 実行前なので反映される）
if (Actor)
{
    Actor->MyProperty = 42;
    Actor->AnotherValue = TEXT("Custom");

    // 3. FinishSpawning で Construction Script 実行 + BeginPlay 準備完了
    Actor->FinishSpawning(SpawnTransform);
}
```

---

## SpawnActor 内部フロー

```
UWorld::SpawnActor(Class, Transform, Params)                    [LevelActor.cpp]
  ├─ StaticConstructObject_Internal(Class, Level, Name, ...)    ← UObject 確保 + CDO コピー
  │    └─ FObjectInitializer::PostConstructInit
  │         └─ T::T() (コンストラクタ)                          ← ここで CreateDefaultSubobject
  │
  ├─ AActor::PostSpawnInitialize(Transform, Owner, Instigator, ..., bDeferConstruction)
  │    ├─ RegisterAllComponents()
  │    │    └─ 各コンポーネント OnRegister / InitializeComponent
  │    ├─ PreInitializeComponents()
  │    ├─ PostInitializeComponents()           ← カスタム初期化フック
  │    └─ if (!bDeferConstruction):
  │         └─ FinishSpawning(Transform)
  │
  └─ return NewActor

AActor::FinishSpawning(Transform, ...)                          [Actor.cpp]
  ├─ ExecuteConstruction(Transform, InstanceDataCache)          ← BP Construction Script
  ├─ PostActorConstruction()
  │    └─ (if World has begun play): DispatchBeginPlay()       ← BeginPlay 発火
  └─ return
```

---

## テンプレート スポーン（FActorSpawnParameters::Template）

既存 Actor の設定をコピーして新しい Actor を生成する場合:

```cpp
// 既存 Actor をテンプレートとして使う
AMyActor* Template = FindActorTemplate();

FActorSpawnParameters Params;
Params.Template = Template;           // CDO の代わりにこの Actor からコピー

AMyActor* Clone = GetWorld()->SpawnActor<AMyActor>(
    AMyActor::StaticClass(),
    SpawnTransform,
    Params
);
```

Blueprint の「ChildActor」や「ConstructionScript での Actor 生成」はこの仕組みを使う。

---

## ESpawnActorCollisionHandlingMethod

| 値 | 説明 |
|----|------|
| `Undefined` | クラスのデフォルト設定に従う |
| `AlwaysSpawn` | 衝突を無視して常にスポーン |
| `AdjustIfPossibleButAlwaysSpawn` | 衝突を避けて位置調整、無理なら強制スポーン |
| `AdjustIfPossibleButDontSpawnIfColliding` | 位置調整可能なら OK、無理なら null |
| `DontSpawnIfColliding` | 衝突があれば null を返す |

---

## Blueprint からのスポーン

| BP ノード | 対応 C++ |
|----------|---------|
| `Spawn Actor from Class` | `SpawnActor<T>(Class, Transform, Params)` |
| `Spawn Actor Deferred` | `SpawnActorDeferred<T>` |
| `Finish Spawning` | `FinishSpawning` |
| `Begin Deferred Actor Spawn By Class` | `SpawnActorDeferred` の旧版 |

---

## よくある失敗

```cpp
// NG: Deferred スポーン後に FinishSpawning を忘れる
AMyActor* Actor = World->SpawnActorDeferred<AMyActor>(...);
Actor->MyProp = 42;
// FinishSpawning を呼ばないと BeginPlay が発火しない！

// NG: SpawnActor の戻り値を検証しない
AMyActor* Actor = World->SpawnActor<AMyActor>(...);
Actor->DoSomething();   // null クラッシュの可能性

// OK
if (AMyActor* Actor = World->SpawnActor<AMyActor>(...))
{
    Actor->DoSomething();
}
```

---

## 関連ドキュメント

- [[a_actor_lifecycle]] — PostSpawnInitialize〜BeginPlay の全体フロー
- [[b_component_model]] — RegisterComponent / OnRegister
- [[Reference/ref_actor_api]] — SpawnActor / FActorSpawnParameters API
- [[../../Core/UObject/Details/d_class_default_object]] — CDO コピーの仕組み
