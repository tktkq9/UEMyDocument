# HLOD ランタイム切り替え・ストリーミング連携

- 上位: [[HLOD/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Public/WorldPartition/HLOD/HLODRuntimeSubsystem.h`

---

## 概要

ランタイム HLOD 管理は **UWorldPartitionHLODRuntimeSubsystem**（旧 `UHLODSubsystem`）が担当する。ストリーミングセルの表示状態変化を監視し、HLOD オブジェクトの可視性を自動制御する。

---

## UWorldPartitionHLODRuntimeSubsystem

```cpp
UCLASS(MinimalAPI)
class UWorldPartitionHLODRuntimeSubsystem : public UWorldSubsystem
{
public:
    // 初期化・解放
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    // HLOD オブジェクトの登録・解除
    void RegisterHLODObject(IWorldPartitionHLODObject* InHLODObject);
    void UnregisterHLODObject(IWorldPartitionHLODObject* InHLODObject);

    // セルの表示状態変化コールバック（StreamingPolicy から呼ばれる）
    void OnCellShown(const UWorldPartitionRuntimeCell* InCell);
    void OnCellHidden(const UWorldPartitionRuntimeCell* InCell);

    // HLOD ↔ 元セルの切り替え可否判定
    bool CanMakeVisible(const UWorldPartitionRuntimeCell* InCell);
    bool CanMakeInvisible(const UWorldPartitionRuntimeCell* InCell);

    // セルに対応する HLOD オブジェクト一覧
    const TArray<IWorldPartitionHLODObject*>& GetHLODObjectsForCell(
        const UWorldPartitionRuntimeCell* InCell) const;

    // HLOD グローバル有効化フラグ（CVar: wp.Runtime.HLOD）
    static bool IsHLODEnabled();
};
```

---

## IWorldPartitionHLODObject — HLOD インターフェース

`AWorldPartitionHLOD` が実装するインターフェース。カスタム HLOD 表現も同一インターフェースで扱える。

```cpp
class IWorldPartitionHLODObject
{
public:
    virtual UObject* GetUObject() const = 0;
    virtual ULevel* GetHLODLevel() const = 0;
    virtual FString GetHLODNameOrLabel() const = 0;

    // ウォームアップ（Nanite/VSM の事前キャッシュ等）が必要か
    virtual bool DoesRequireWarmup() const = 0;
    virtual TSet<UObject*> GetAssetsToWarmup() const = 0;

    // 可視性制御
    virtual void SetVisibility(bool bIsVisible) = 0;

    // ソースセルの GUID（どのセルの HLOD か）
    virtual const FGuid& GetSourceCellGuid() const = 0;

    // スタンドアロン HLOD（セルに紐づかない）
    virtual bool IsStandalone() const = 0;
};
```

---

## セル ↔ HLOD 切り替えフロー

```mermaid
stateDiagram-v2
    [*] --> HLOD表示 : 初期状態（セル未ロード）

    HLOD表示 --> 遷移中 : OnCellShown(Cell)
    遷移中 --> 元アクタ表示 : CanMakeVisible = true
    遷移中 --> HLOD表示 : CanMakeVisible = false（ロード未完了）

    元アクタ表示 --> 遷移中2 : OnCellHidden(Cell)
    遷移中2 --> HLOD表示 : CanMakeInvisible = true
```

### CanMakeVisible / CanMakeInvisible の判定

```cpp
// セルの全アクタがロード完了したときのみ true
bool CanMakeVisible(const UWorldPartitionRuntimeCell* InCell);

// HLOD が存在し、かつウォームアップ済みのとき true
bool CanMakeInvisible(const UWorldPartitionRuntimeCell* InCell);
```

ウォームアップ（`DoesRequireWarmup() = true`）が必要な HLOD の場合、Nanite メッシュ等の GPU リソースがキャッシュされるまで元セルを非表示にしない。

---

## デリゲート

```cpp
// HLOD オブジェクト登録イベント
FWorldPartitionHLODObjectRegisteredEvent& OnHLODObjectRegisteredEvent();
FWorldPartitionHLODObjectUnregisteredEvent& OnHLODObjectUnregisteredEvent();

// セルの全 HLOD を走査するイベント
FWorldPartitionHLODForEachHLODObjectInCellEvent& GetForEachHLODObjectInCellEvent();
```

---

## ExternalStreamingObject との連携

DLC / ContentBundle の外部ストリーミングオブジェクトが注入される際にも HLOD サブシステムが対応。

```cpp
// 外部ストリーミングオブジェクトの注入・削除コールバック
void OnExternalStreamingObjectInjected(URuntimeHashExternalStreamingObjectBase*);
void OnExternalStreamingObjectRemoved(URuntimeHashExternalStreamingObjectBase*);
```

---

## HLOD ウォームアップシステム

**Nanite** や **Virtual Shadow Map（VSM）** を使った HLOD は、初回表示時にGPU でのキャッシュビルドが必要。  
`UHLODResourcesResidencySceneViewExtension` がフレームごとに資産の常駐状態を監視し、準備完了まで HLOD を表示しないよう制御する。

---

## CVars

| CVar | デフォルト | 説明 |
|------|----------|------|
| `wp.Runtime.HLOD` | `1` | HLOD 全体の有効/無効 |
| `wp.Runtime.HLODWarmupEnabled` | `1` | ウォームアップシステムの有効/無効 |
| `wp.Runtime.HLODWarmupNumFrames` | `5` | ウォームアップに必要なフレーム数 |

---

## 関連クラス

| クラス | 役割 |
|-------|------|
| `AWorldPartitionHLOD` | HLOD アクタ本体 |
| `UHLODLayer` | ビルド設定アセット |
| `UHLODBuilder` | メッシュ生成ロジック |
| `UWorldPartitionRuntimeCell` | ストリーミングセル（HLOD の対象） |

---

## コード実行フロー

### エントリポイント

```
[初期化 — サブシステム登録]
UWorldPartitionHLODRuntimeSubsystem::Initialize(Collection)
  ├─ UWorldPartition::OnWorldPartitionInitializedDelegate を購読
  ├─ OnCellShownDelegate / OnCellHiddenDelegate を購読
  └─ HLOD アクタのスポーン時に RegisterHLODObject() が呼ばれ CellToHLODMap に登録

[ランタイム — 可視性制御]
StreamingPolicy::PostUpdateStreamingStateInternal_GameThread()
  └─ Cell->Activate() → AddToWorld()
       └─ FLevelStreamingDelegates::OnCellShownDelegate
            └─ UWorldPartitionHLODRuntimeSubsystem::OnCellShown(Cell)
                 └─ CanMakeInvisible(Cell) 判定
                      ├─ 全 HLOD の IsReadyForWarmup() チェック
                      └─ 元アクタの LoadedLevel が存在するか
                 └─ GetHLODObjectsForCell(Cell):
                      └─ for each IWorldPartitionHLODObject:
                           └─ HLODObject->SetVisibility(false)

  └─ Cell->Deactivate() → RemoveFromWorld()
       └─ OnCellHiddenDelegate
            └─ UWorldPartitionHLODRuntimeSubsystem::OnCellHidden(Cell)
                 └─ CanMakeVisible(Cell) 判定
                 └─ for each IWorldPartitionHLODObject:
                      └─ HLODObject->SetVisibility(true)

[ウォームアップ — SceneViewExtension]
UHLODResourcesResidencySceneViewExtension::BeginRenderViewFamily()
  └─ for each HLOD を購読中の RequestContext:
       └─ Nanite::GStreamingManager->RequestPrefetch(HLODMesh)
       └─ VirtualShadowMap キャッシュ準備チェック
       └─ N フレーム常駐したら IsReadyForWarmup=true
```

### フロー詳細

1. **サブシステム初期化** — `UWorldPartitionHLODRuntimeSubsystem::Initialize` はワールド初期化時に `UWorldPartition` のイベントを購読。HLOD アクタが `PostRegisterAllComponents()` で `RegisterHLODObject()` を呼び、`CellToHLODMap` に登録される。
2. **セル表示イベント** — `OnCellShown()` はストリーミングポリシーから通知される。`CellToHLODMap` から対応する HLOD 群を取得し、元セルの準備が整うまで HLOD を表示維持（ポップイン防止）。
3. **CanMakeInvisible 判定** — 元セルの `LoadedLevel` が有効かつ全アクタコンポーネントが登録済みなら true。`bBlockOnSlowLoading=true` のセルはロード完了までブロック。
4. **SetVisibility 一括適用** — `GetHLODObjectsForCell()` が返す HLOD リストに対し `SetVisibility(false)` を適用。単一セルが複数 LOD 階層の HLOD に対応する場合もあるので全て切替。
5. **セル非表示イベント** — `OnCellHidden()` は逆方向の切替。HLOD にウォームアップが必要（`DoesRequireWarmup()=true`）なら `UHLODResourcesResidencySceneViewExtension` が `wp.Runtime.HLODWarmupNumFrames` 分の常駐を確認してから表示。
6. **Nanite/VSM ウォームアップ** — `BeginRenderViewFamily` 内で `Nanite::GStreamingManager::RequestPrefetch` を呼び、仮想テクスチャ・VSM キャッシュもリクエスト。準備完了で HLOD 表示許可フラグが立つ。
7. **外部ストリーミング連携** — ContentBundle / DLC による `URuntimeHashExternalStreamingObjectBase` 注入時は `OnExternalStreamingObjectInjected()` が追加 HLOD を登録。削除時は `UnregisterHLODObject()` で除外。
8. **CVar による無効化** — `wp.Runtime.HLOD 0` で `IsHLODEnabled()=false`、全 HLOD が強制非表示。`wp.Runtime.HLODWarmupEnabled 0` はウォームアップ待機を省略して即切替。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `UWorldPartitionHLODRuntimeSubsystem::Initialize` | `HLOD/HLODRuntimeSubsystem.cpp` | サブシステム開始 |
| `UWorldPartitionHLODRuntimeSubsystem::OnCellShown` | `HLOD/HLODRuntimeSubsystem.cpp` | セル→HLOD切替 |
| `UWorldPartitionHLODRuntimeSubsystem::OnCellHidden` | `HLOD/HLODRuntimeSubsystem.cpp` | HLOD→セル切替 |
| `UWorldPartitionHLODRuntimeSubsystem::CanMakeVisible/Invisible` | `HLOD/HLODRuntimeSubsystem.cpp` | 切替可否判定 |
| `IWorldPartitionHLODObject::SetVisibility` | `HLOD/HLODActor.cpp` | 可視性適用 |
| `UHLODResourcesResidencySceneViewExtension` | `HLOD/HLODResourcesResidencySceneViewExtension.cpp` | ウォームアップ監視 |
| `FLevelStreamingDelegates::OnCellShown/Hidden` | `Engine/LevelStreamingDelegates.h` | セルイベント |
