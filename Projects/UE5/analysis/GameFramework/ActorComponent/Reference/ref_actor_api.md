# AActor 主要 API リファレンス

- 上位: [[ActorComponent/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Classes/GameFramework/Actor.h`

---

## ライフサイクル仮想関数

| 関数 | 説明 | 呼ばれるタイミング |
|------|------|------------------|
| `OnConstruction(Transform)` | コンストラクションスクリプト相当 | 配置・Transform 変更時 |
| `PreInitializeComponents()` | Component 初期化前 | PostSpawnInitialize 内 |
| `PostInitializeComponents()` | Component 初期化後 | PostSpawnInitialize 内 |
| `BeginPlay()` | ゲーム開始時 | World::BeginPlay から |
| `Tick(DeltaTime)` | 毎フレーム | TickGroup に従い |
| `EndPlay(Reason)` | 終了時 | Destroy / LevelUnload 等 |
| `Destroyed()` | 破棄直前 | MarkPendingKill 前 |

---

## 位置・変換

```cpp
// 取得
FVector  GetActorLocation() const;
FRotator GetActorRotation() const;
FQuat    GetActorQuat() const;
FVector  GetActorScale3D() const;
FTransform GetActorTransform() const;

FVector GetActorForwardVector() const;  // X 軸方向
FVector GetActorRightVector()   const;  // Y 軸方向
FVector GetActorUpVector()      const;  // Z 軸方向

// 設定（bSweep=true でコリジョンあり移動）
bool SetActorLocation(const FVector& Loc, bool bSweep=false,
                      FHitResult* Hit=nullptr,
                      ETeleportType Teleport=ETeleportType::None);

bool SetActorRotation(FRotator Rot,
                      ETeleportType Teleport=ETeleportType::None);

bool SetActorLocationAndRotation(FVector Loc, FRotator Rot,
                                 bool bSweep=false,
                                 FHitResult* Hit=nullptr,
                                 ETeleportType Teleport=ETeleportType::None);

void SetActorTransform(const FTransform& NewTransform,
                       bool bSweep=false,
                       FHitResult* Hit=nullptr,
                       ETeleportType Teleport=ETeleportType::None);

// 相対移動
void AddActorWorldOffset(FVector DeltaLocation, bool bSweep=false, ...);
void AddActorWorldRotation(FRotator DeltaRotation, bool bSweep=false, ...);

// テレポート（コリジョン判定なし高速移動）
bool TeleportTo(const FVector& Dest, const FRotator& DestRot,
                bool bIsATest=false, bool bNoCheck=false);
```

---

## 生存・破棄

```cpp
// 生存確認
bool IsValid(const AActor* Actor);   // nullptr && !IsPendingKill
bool IsPendingKillPending() const;

// 破棄
void Destroy(bool bNetForce=false, bool bShouldModifyLevel=true);

// 自動破棄タイマー
void SetLifeSpan(float InLifespan);  // 0.f = 無効
float GetLifeSpan() const;
```

---

## 所有・階層

```cpp
AActor* GetOwner() const;
void    SetOwner(AActor* NewOwner);

// 子 Actor 一覧
void GetAttachedActors(TArray<AActor*>& OutActors,
                       bool bResetArray=true,
                       bool bRecursivelyIncludeAttachedActors=false) const;

// Root Component
USceneComponent* GetRootComponent() const;
void SetRootComponent(USceneComponent* NewRootComp);
```

---

## コンポーネント取得

```cpp
// 型で最初の 1 つを取得
template<class T>
T* FindComponentByClass() const;

UActorComponent* FindComponentByClass(TSubclassOf<UActorComponent> Class) const;

// 型で全取得
template<class T>
void GetComponents(TArray<T*>& OutComponents, bool bIncludeChildActors=false) const;

TArray<UActorComponent*> K2_GetComponentsByClass(
    TSubclassOf<UActorComponent> Class) const;

// タグで取得
UActorComponent* FindComponentByTag(TSubclassOf<UActorComponent> Class,
                                    FName Tag) const;
```

---

## ネットワーク

```cpp
bool HasAuthority() const;   // サーバー or 単独プレイ → true
bool IsLocallyControlled() const; // ローカルコントローラー → true

bool GetIsReplicated() const;
void SetReplicates(bool bInReplicates);
void SetReplicateMovement(bool bInReplicateMovement);

virtual void GetLifetimeReplicatedProps(
    TArray<FLifetimeProperty>& OutLifetimeProps) const override;
```

---

## 表示・コリジョン

```cpp
void SetActorHiddenInGame(bool bNewHidden);
bool IsHidden() const;

void SetActorEnableCollision(bool bNewActorEnableCollision);
bool GetActorEnableCollision() const;

void SetActorTickEnabled(bool bEnabled);
bool IsActorTickEnabled() const;
```

---

## オーバーラップ・ヒット

```cpp
void GetOverlappingActors(TArray<AActor*>& Out,
                          TSubclassOf<AActor> ClassFilter=nullptr) const;

void GetOverlappingComponents(TArray<UPrimitiveComponent*>& Out) const;

bool IsOverlappingActor(const AActor* Other) const;
```

---

## タグ・ラベル

```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category=Actor)
TArray<FName> Tags;

bool ActorHasTag(FName Tag) const;
```

---

## ワールド・タイマー

```cpp
UWorld* GetWorld() const;

FTimerManager& GetWorldTimerManager() const;
// → GetWorldTimerManager().SetTimer(Handle, Func, Rate, bLoop);
```

---

## 関連ドキュメント

- [[Details/a_actor_lifecycle]] — ライフサイクル詳細
- [[Details/d_actor_spawning]] — SpawnActor パラメータ
- [[ref_component_api]] — UActorComponent API
