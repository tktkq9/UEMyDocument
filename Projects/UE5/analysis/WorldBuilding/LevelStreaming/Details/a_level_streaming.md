# ULevelStreaming ライフサイクル・非同期ロード

- 上位: [[LevelStreaming/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Classes/Engine/LevelStreaming.h`

---

## 概要

`ULevelStreaming` はサブレベル（`ULevel`）の非同期ロード・アンロード・表示/非表示を管理する抽象基底クラス。UE4 時代からある機能で、UE5 では World Partition が内部で `UWorldPartitionLevelStreamingDynamic`（派生クラス）を使ってセルをストリーミングする。

---

## クラス階層

```mermaid
classDiagram
    class ULevelStreaming {
        <<abstract>>
        +bShouldBeLoaded: bool
        +bShouldBeVisible: bool
        +LevelTransform: FTransform
        +LoadedLevel: ULevel*
        +SetShouldBeLoaded(bool)
        +SetShouldBeVisible(bool)
        +GetLevelStreamingState() ELevelStreamingState
    }
    class ULevelStreamingDynamic {
        +LoadLevelInstance(Name, World, Transform, Success)
        +LoadLevelInstanceBySoftObjectPtr(...)
    }
    class ULevelStreamingAlwaysLoaded
    class ULevelStreamingPersistent
    class UWorldPartitionLevelStreamingDynamic {
        <<WP 内部実装>>
    }

    ULevelStreaming <|-- ULevelStreamingDynamic
    ULevelStreaming <|-- ULevelStreamingAlwaysLoaded
    ULevelStreaming <|-- ULevelStreamingPersistent
    ULevelStreamingDynamic <|-- UWorldPartitionLevelStreamingDynamic
```

---

## ELevelStreamingState — ストリーミング状態

```cpp
enum class ELevelStreamingState : uint8
{
    Removed,          // ストリーミングオブジェクトが除去済み
    Unloaded,         // アンロード状態
    FailedToLoad,     // ロード失敗
    Loading,          // ロード中（非同期）
    LoadedNotVisible, // ロード済み・非表示
    MakingVisible,    // 表示処理中（AddToWorld 中）
    LoadedVisible,    // ロード済み・表示中
    MakingInvisible,  // 非表示処理中（RemoveFromWorld 中）
};
```

---

## ELevelStreamingTargetState — 目標状態

```cpp
enum class ELevelStreamingTargetState : uint8
{
    Unloaded,            // アンロード（最終目標）
    UnloadedAndRemoved,  // アンロード後にオブジェクトも削除
    LoadedNotVisible,    // ロード済み・非表示
    LoadedVisible,       // ロード済み・表示
};
```

---

## 主要プロパティ

| プロパティ | 型 | BP公開 | 説明 |
|-----------|-----|--------|------|
| `WorldAsset` | `TSoftObjectPtr<UWorld>` | 読み取り | ロード対象のワールドアセット |
| `LevelTransform` | `FTransform` | 読み書き | ロード後のアクタ変換 |
| `StreamingPriority` | `int32` | Setter | ロード優先度（高いほど優先） |
| `bShouldBeLoaded` | `bool` | Setter/Getter | ロード要求フラグ |
| `bShouldBeVisible` | `bool` | Setter | 可視性フラグ |
| `bShouldBlockOnLoad` | `bool` | 読み書き | 同期ロード（ブロッキング） |
| `bShouldBlockOnUnload` | `bool` | 読み書き | 同期アンロード |
| `bDisableDistanceStreaming` | `bool` | 読み書き | 距離ベースの自動制御を無効化 |
| `EditorStreamingVolumes` | `TArray<ALevelStreamingVolume*>` | — | ボリュームトリガーリスト |

---

## 主要メソッド

### 状態制御

```cpp
// ロード要求（true でロード開始）
UFUNCTION(BlueprintSetter)
void SetShouldBeLoaded(bool bInShouldBeLoaded);

// 可視性制御（true でワールドに追加・AddToWorld）
UFUNCTION(BlueprintSetter)
void SetShouldBeVisible(bool bInShouldBeVisible);

// 優先度設定
UFUNCTION(BlueprintSetter)
void SetPriority(int32 NewPriority);

// 一時的な優先度オーバーライド
void SetPriorityOverride(int32 PriorityOverride);
void ResetPriorityOverride();
```

### 状態確認

```cpp
// 現在のストリーミング状態
ELevelStreamingState GetLevelStreamingState() const;

// ロードリクエスト保留中か
bool HasLoadRequestPending() const;

// ロード済みレベルが存在するか
bool HasLoadedLevel() const;

// アンロード・削除要求フラグ
UFUNCTION(BlueprintPure, Category = LevelStreaming)
bool GetIsRequestingUnloadAndRemoval() const;

UFUNCTION(BlueprintCallable, Category = LevelStreaming)
void SetIsRequestingUnloadAndRemoval(bool bInIsRequestingUnloadAndRemoval);
```

---

## 非同期ロードフロー

```mermaid
sequenceDiagram
    participant Code as Game Code
    participant LS as ULevelStreaming
    participant AM as AsyncLoader
    participant World as UWorld

    Code->>LS: SetShouldBeLoaded(true)
    Note over LS: CurrentState = Loading
    LS->>AM: RequestAsyncLoad(PackagePath)
    AM-->>LS: OnLoadingFinished() コールバック
    Note over LS: CurrentState = LoadedNotVisible

    Code->>LS: SetShouldBeVisible(true)
    Note over LS: CurrentState = MakingVisible
    LS->>World: AddToWorld(Level, Transform)
    World-->>LS: 完了
    Note over LS: CurrentState = LoadedVisible
```

### 非同期ロードの実装詳細

`AsyncRequestIDs` に非同期リクエスト ID を格納し、`OnLoadingFinished()` で ID をクリア。ロード完了は次フレームの `UWorld::UpdateLevelStreaming()` で検出される。

---

## デリゲート（BP アサイン可能）

```cpp
// ロード完了イベント
UPROPERTY(BlueprintAssignable)
FLevelStreamingLoadedStatus OnLevelLoaded;

// アンロード完了イベント
UPROPERTY(BlueprintAssignable)
FLevelStreamingLoadedStatus OnLevelUnloaded;

// 表示完了イベント（AddToWorld 完了）
UPROPERTY(BlueprintAssignable)
FLevelStreamingVisibilityStatus OnLevelShown;

// 非表示完了イベント（RemoveFromWorld 完了）
UPROPERTY(BlueprintAssignable)
FLevelStreamingVisibilityStatus OnLevelHidden;
```

---

## ALevelStreamingVolume — ボリュームトリガー

```cpp
UCLASS(MinimalAPI)
class ALevelStreamingVolume : public AVolume
{
    // 対象レベルストリーミングオブジェクト
    UPROPERTY(EditAnywhere, Category = LevelStreaming)
    TArray<TLazyObjectPtr<class ULevelStreaming>> StreamingLevels;

    // ロードのみ（表示しない）
    UPROPERTY(EditAnywhere, Category = LevelStreaming)
    bool bDisabled;
};
```

プレイヤーがボリュームに入ると自動的に `SetShouldBeLoaded(true)` / `SetShouldBeVisible(true)` が呼ばれる。

---

## CVars

| CVar | 説明 |
|------|------|
| `s.LevelStreamingActorsUpdateTimeLimit` | AddToWorld でアクタ追加に使える時間制限（ms）|
| `s.UnregisterComponentsTimeLimit` | RemoveFromWorld でのコンポーネント登録解除制限 |
| `s.PriorityLevelStreamingActorsUpdateExtraTime` | 高優先度レベルへの追加時間 |

---

## コード実行フロー

### エントリポイント

```
[毎フレーム駆動]
UWorld::Tick()
  └─ UWorld::UpdateLevelStreaming()                        [World.cpp:4932]
       ├─ FStreamingLevelsToConsider を走査
       └─ for each ULevelStreaming:
            └─ ULevelStreaming::UpdateStreamingState()     [LevelStreaming.cpp:992]
                 ├─ DetermineTargetState()
                 │    └─ bShouldBeLoaded / bShouldBeVisible から ELevelStreamingTargetState 決定
                 ├─ switch (TargetState vs CurrentState):
                 │    ├─ → Loading:
                 │    │    └─ RequestLevel(World, PackageName, DataReleaseType)  [LevelStreaming.cpp:1551]
                 │    │         ├─ LoadPackageAsync(PackagePath, CompletionCallback)
                 │    │         │    └─ AsyncRequestIDs に ID 保存
                 │    │         └─ AsyncLoadCallback(Pkg, Result):
                 │    │              ├─ UWorld* LoadedWorld = FindWorld(Pkg)
                 │    │              ├─ ULevel* NewLevel = LoadedWorld->PersistentLevel
                 │    │              └─ SetLoadedLevel(NewLevel)
                 │    │                   └─ OnLevelLoaded.Broadcast()
                 │    ├─ → MakingVisible:
                 │    │    └─ ULevel::AddToWorld(World, Transform)
                 │    │         ├─ インクリメンタルに actors を UWorld に追加
                 │    │         ├─ Component 登録 / PhysicsState 作成
                 │    │         └─ OnLevelShown.Broadcast()
                 │    ├─ → MakingInvisible:
                 │    │    └─ ULevel::RemoveFromWorld()
                 │    │         ├─ Component 登録解除
                 │    │         └─ OnLevelHidden.Broadcast()
                 │    └─ → Unloaded:
                 │         └─ UPackage::MarkAsUnreachable → GC で解放
                 └─ PostUpdateStreamingState() で次フレーム用フラグ整理
```

### フロー詳細

1. **UpdateLevelStreaming 駆動** — `UWorld::Tick` から毎フレーム呼ばれ、登録済み `ULevelStreaming` を `FStreamingLevelsToConsider` 経由で走査。`bShouldBeLoaded`/`bShouldBeVisible` 変更があったものだけキューされる最適化あり。
2. **TargetState 決定** — 現在の状態と要求フラグから `ELevelStreamingTargetState` を計算。`bShouldBeLoaded=false && LoadedLevel!=nullptr` なら `Unloaded` を目指す。
3. **非同期ロード要求** — `RequestLevel()` が `LoadPackageAsync` に委譲。`bShouldBlockOnLoad=true` なら `FlushAsyncLoading()` で同期化（フレームドロップ覚悟）。返却される `FPackageRequestID` は `AsyncRequestIDs` に記録。
4. **ロード完了コールバック** — `AsyncLoadCallback(Pkg, Result)` が完了後に呼ばれ、パッケージ内の `UWorld` を探索し `PersistentLevel` を取得。`SetLoadedLevel()` で保持し `OnLevelLoaded` を Broadcast。
5. **AddToWorld インクリメンタル** — `bShouldBeVisible=true` なら `ULevel::AddToWorld()` が呼ばれ、`s.LevelStreamingActorsUpdateTimeLimit` で指定された時間内でアクタ登録を分割処理。複数フレームに跨ぐ可能性あり（`MakingVisible` 状態の間）。
6. **RemoveFromWorld** — 逆方向は `RemoveFromWorld()` で `s.UnregisterComponentsTimeLimit` 内でコンポーネント登録解除。終了で `LoadedNotVisible` に戻る。
7. **アンロード** — `TargetState=Unloaded` なら `LoadedLevel` の `UPackage` を `MarkAsGarbage` し、次の GC で解放。`bShouldBlockOnUnload=true` なら同期 GC をトリガー。
8. **優先度制御** — `StreamingPriority` は `LoadPackageAsync` のロード順に反映。`SetPriorityOverride()` で一時的に引き上げ（例: プレイヤー近傍のセル）。
9. **WP との関係** — WorldPartition セルは `UWorldPartitionLevelStreamingDynamic` を生成し、ストリーミングポリシーから `SetShouldBeLoaded`/`SetShouldBeVisible` を呼ぶだけ。以降の処理は全て本ファイルのフロー。
10. **ボリュームトリガー** — `ALevelStreamingVolume` 内にプレイヤーが入ると `UWorld::ProcessLevelStreamingVolumes()` が対応する `ULevelStreaming` のフラグを立てる。WP と併用も可能。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `UWorld::UpdateLevelStreaming` | `World.cpp:4932` | 毎フレーム駆動 |
| `ULevelStreaming::UpdateStreamingState` | `LevelStreaming.cpp:992` | 状態差分→アクション |
| `ULevelStreaming::RequestLevel` | `LevelStreaming.cpp:1551` | 非同期ロード要求 |
| `ULevelStreaming::AsyncLoadCallback` | `LevelStreaming.cpp` | ロード完了処理 |
| `ULevel::AddToWorld` | `Level.cpp` | ワールド統合（インクリメンタル） |
| `ULevel::RemoveFromWorld` | `Level.cpp` | ワールド除去 |
| `UWorld::ProcessLevelStreamingVolumes` | `World.cpp` | ボリュームトリガー |
| `FLevelStreamingLoadedStatus` | `LevelStreaming.h` | ロード/アンロードデリゲート |
