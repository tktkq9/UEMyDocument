# UActorComponent / USceneComponent API リファレンス

- 上位: [[ActorComponent/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Classes/Components/ActorComponent.h`, `SceneComponent.h`

---

## UActorComponent — ライフサイクル仮想関数

| 関数 | 説明 |
|------|------|
| `InitializeComponent()` | `bWantsInitializeComponent=true` のとき BeginPlay 前に呼ばれる |
| `BeginPlay()` | ゲーム開始時 |
| `TickComponent(DeltaTime, TickType, TickFunc)` | 毎フレーム（有効時のみ） |
| `EndPlay(Reason)` | 終了時 |
| `UninitializeComponent()` | InitializeComponent の対 |
| `OnRegister()` | ワールドへの登録時 |
| `OnUnregister()` | 登録解除時 |

---

## UActorComponent — 有効化・Tick

```cpp
// 有効化
void SetActive(bool bNewActive, bool bReset=false);
void Activate(bool bReset=false);
void Deactivate();
void ToggleActive();
bool IsActive() const;

// Tick 制御
void SetComponentTickEnabled(bool bEnabled);
bool IsComponentTickEnabled() const;
void SetComponentTickInterval(float TickInterval);
float GetComponentTickInterval() const;

// Tick グループ
PrimaryComponentTick.TickGroup = TG_PrePhysics;  // 設定はプロパティ直接操作
```

---

## UActorComponent — 所有・登録

```cpp
AActor*       GetOwner() const;
UWorld*       GetWorld() const;
ENetRole      GetOwnerRole() const;  // ROLE_Authority / ROLE_AutonomousProxy / ROLE_SimulatedProxy

void RegisterComponent();    // ワールドに登録
void UnregisterComponent();  // 登録解除
void DestroyComponent(bool bPromoteChildren=false);
bool IsRegistered() const;
bool IsBeingDestroyed() const;

// ネット
bool GetIsReplicated() const;
void SetIsReplicated(bool ShouldReplicate);
```

---

## USceneComponent — 変換（相対座標）

```cpp
// 取得
FVector   GetRelativeLocation() const;
FRotator  GetRelativeRotation() const;
FVector   GetRelativeScale3D() const;

// 設定
void SetRelativeLocation(FVector NewLocation, bool bSweep=false, ...);
void SetRelativeRotation(FRotator NewRotation, bool bSweep=false, ...);
void SetRelativeScale3D(FVector NewScale3D);
void SetRelativeLocationAndRotation(FVector Loc, FRotator Rot, bool bSweep=false, ...);

// 相対移動
void AddRelativeLocation(FVector DeltaLocation, bool bSweep=false, ...);
void AddRelativeRotation(FRotator DeltaRotation, bool bSweep=false, ...);
```

---

## USceneComponent — 変換（ワールド座標）

```cpp
// 取得
FVector    GetComponentLocation() const;
FRotator   GetComponentRotation() const;
FQuat      GetComponentQuat() const;
FVector    GetComponentScale() const;
FTransform GetComponentTransform() const;

FVector    GetForwardVector() const;  // ローカル X 軸（ワールド空間）
FVector    GetRightVector()   const;
FVector    GetUpVector()      const;

// 設定
void SetWorldLocation(FVector NewLocation, bool bSweep=false, ...);
void SetWorldRotation(FRotator NewRotation, bool bSweep=false, ...);
void SetWorldScale3D(FVector NewScale);
void SetWorldLocationAndRotation(FVector Loc, FRotator Rot, bool bSweep=false, ...);
```

---

## USceneComponent — アタッチ

```cpp
// アタッチ
bool AttachToComponent(USceneComponent* Parent,
                        const FAttachmentTransformRules& Rules,
                        FName SocketName = NAME_None);

// デタッチ
void DetachFromComponent(const FDetachmentTransformRules& Rules);

// アタッチ状態
bool IsAttachedTo(const USceneComponent* TestComp) const;
USceneComponent* GetAttachParent() const;
FName GetAttachSocketName() const;

// 子コンポーネント
const TArray<TObjectPtr<USceneComponent>>& GetAttachChildren() const;
```

### FAttachmentTransformRules 定数

```cpp
// 相対位置を維持（デフォルト的な使い方）
FAttachmentTransformRules::KeepRelativeTransform

// ワールド位置を維持
FAttachmentTransformRules::KeepWorldTransform

// ゼロにスナップ
FAttachmentTransformRules::SnapToTargetNotIncludingScale
FAttachmentTransformRules::SnapToTargetIncludingScale
```

---

## USceneComponent — 表示・ソケット

```cpp
void SetVisibility(bool bNewVisibility,
                   EVisibilityPropagation PropagateToChildren = EVisibilityPropagation::NoPropagation);
bool IsVisible() const;

// ソケット
FTransform GetSocketTransform(FName SocketName,
                               ERelativeTransformSpace TransformSpace = RTS_World) const;
bool DoesSocketExist(FName SocketName) const;
TArray<FName> GetAllSocketNames() const;
```

---

## 主要プロパティ（UPROPERTY）

```cpp
// UActorComponent
UPROPERTY(EditAnywhere, BlueprintReadOnly)
bool bAutoActivate;         // 生成時に自動 Activate

UPROPERTY(EditDefaultsOnly)
bool bWantsInitializeComponent; // InitializeComponent() を呼ぶか

// USceneComponent
UPROPERTY(EditAnywhere, BlueprintReadWrite)
FVector   RelativeLocation;
FRotator  RelativeRotation;
FVector   RelativeScale3D;

UPROPERTY(EditAnywhere, BlueprintReadWrite)
bool bAbsoluteLocation;  // 親の位置を無視してワールド座標固定
bool bAbsoluteRotation;
bool bAbsoluteScale;
```

---

## 関連ドキュメント

- [[Details/b_component_model]] — コンポーネントモデル詳細
- [[Details/c_ticking]] — TickFunction 設定
- [[ref_actor_api]] — AActor API
