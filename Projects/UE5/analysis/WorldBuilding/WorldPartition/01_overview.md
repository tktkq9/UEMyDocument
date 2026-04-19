# WorldPartition 概要

- 上位: [[01_worldbuilding_overview]]
- 関連: [[HLOD/01_overview]] | [[DataLayer/01_overview]] | [[LevelStreaming/01_overview]]
- ソース: `Engine/Source/Runtime/Engine/Public/WorldPartition/`

---

## WorldPartition とは

UE5 のワールドを **空間ハッシュベースのセルに自動分割** し、カメラ位置に基づいて必要なセルだけをストリーミングする仕組み。UE4 の手動サブレベル管理を置き換え、大規模オープンワールドの制作を実用的にする。

---

## アーキテクチャ

```mermaid
graph TD
    WP[UWorldPartition] -->|owns| Hash[UWorldPartitionRuntimeHash]
    Hash <|-- SpatialHash[UWorldPartitionRuntimeSpatialHash]
    Hash <|-- HashSet[UWorldPartitionRuntimeHashSet]
    Hash -->|creates| Cell[UWorldPartitionRuntimeCell]
    Cell -->|inherits| LSCell[UWorldPartitionRuntimeLevelStreamingCell]
    WP -->|owns| Policy[UWorldPartitionStreamingPolicy]
    Policy -->|uses| Source[FWorldPartitionStreamingSource]
    Policy -->|triggers| LS[ULevelStreamingDynamic]
    WP -->|contains| ADC[UActorDescContainer]
    ADC -->|stores| AD[FWorldPartitionActorDesc]
```

---

## 主要クラス

| クラス | 役割 |
|-------|------|
| `UWorldPartition` | WP の中核。初期化・Tick・ランタイムハッシュ/ポリシー管理 |
| `UWorldPartitionRuntimeSpatialHash` | 空間ハッシュでワールドをグリッド分割 |
| `UWorldPartitionRuntimeCell` | 1 セルの単位。バウンド・含有アクタリスト |
| `UWorldPartitionStreamingPolicy` | ストリーミング判定ロジック（ソース位置→セル活性化） |
| `FWorldPartitionStreamingSource` | ストリーミングの基準点（カメラ・カスタム） |
| `FWorldPartitionActorDesc` | エディタ時のアクタ記述子（ロードなしでメタデータ参照） |
| `UWorldPartitionSubsystem` | ランタイムサブシステム（BP 公開） |
| `UWorldPartitionBlueprintLibrary` | BP 向けユーティリティ関数群 |

---

## Details

| ドキュメント | 内容 |
|------------|------|
| [[Details/a_spatial_hash]] | 空間ハッシュ・グリッド設定・セル分割アルゴリズム |
| [[Details/b_streaming_policy]] | ストリーミングポリシー・ソース・ロード/アンロード |
| [[Details/c_actor_desc]] | ActorDesc・ActorDescView・エディタ連携 |
| [[Details/d_runtime_cell]] | RuntimeCell・CellBounds・ContentBundle |

---

## コード実行フロー

### エントリポイント

```
UWorldPartition::Tick()                                     [WorldPartition.cpp:1797]
  └─ (Policy 駆動は UWorldPartitionSubsystem から)

UWorldPartitionSubsystem::UpdateStreamingStateInternal()    [WorldPartitionSubsystem.cpp:1185]
  ├─ UpdateStreamingSources()                              ← Provider から基準点収集
  ├─ UpdateServerClientsVisibleLevelNames()                ← DS 用
  └─ for each RegisteredWorldPartition:
       └─ StreamingPolicy->UpdateStreamingState()          [WorldPartitionStreamingPolicy.cpp:273]
            ├─ WaitForAsyncUpdateStreamingState()          ← 前フレーム非同期タスクの結果回収
            ├─ ComputeUpdateStreamingHash()                ← 入力変化チェック
            ├─ UpdateStreamingStateInternal()              [WorldPartitionStreamingPolicy.cpp:392]
            │    ├─ RuntimeHash->GetStreamingCells()       ← R-tree 空間検索
            │    ├─ 差分計算 → TargetState
            │    └─ PostUpdateStreamingStateInternal_GameThread()
            │         └─ Cell->SetStreamingStatus()
            │              └─ ULevelStreamingDynamic::SetShouldBeLoaded()
            └─ OnStreamingStateUpdated.Broadcast()

セル可視化完了時:
UWorldPartition::OnCellShown()                             [WorldPartition.cpp:1974]
  ├─ HLODRuntimeSubsystem->OnCellShown()
  └─ StreamingPolicy->OnCellShown()
```

### フロー詳細

1. **非同期更新か判定** — `bCanUpdateAsync` (DedicatedServer でない && 初期化済み && BlockTill 中でない) なら `AsyncUpdateTaskState=Pending` にして `OnStreamingStateUpdated` 経由で `UE::Tasks::Launch` する（`WorldPartitionStreamingPolicy.cpp:376–389`）。
2. **差分早期 return** — `ComputeUpdateStreamingHash()` が前フレームと同一ハッシュで 2 フレーム連続なら `bShouldSkipUpdate=true` で return。CPU 節約（`WorldPartitionStreamingPolicy.cpp:363–371`）。
3. **セル取得** — `UWorldPartitionRuntimeHashSet::GetStreamingCells()` が各ソースについて `FRuntimePartitionStreamingData` の R-tree を走査し、該当セルを `FrameActivateCells` / `FrameLoadCells` に収集（[[Details/a_spatial_hash]]）。
4. **TargetState 計算** — 現在の `ActivatedCells` / `LoadedCells` との差分で 4 種遷移リストを構築（[[Details/b_streaming_policy]]）。
5. **インクリメンタル処理** — 複数 WorldPartition がある場合、`GUpdateStreamingStateTimeLimit` によるタイムバジェットで `IncrementalUpdateStreamingState()` が分割処理（`WorldPartitionSubsystem.cpp:1233–1246`）。
6. **ActorDesc による事前知識** — セル選別はロード前に `FWorldPartitionActorDesc` のバウンドを使うため、アクタを実ロードせずに判定可能（[[Details/c_actor_desc]]）。

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|---------|------|
| `UWorldPartition::Tick` | `WorldPartition.cpp:1797` | エディタハッシュ更新など補助処理 |
| `UWorldPartitionSubsystem::UpdateStreamingStateInternal` | `WorldPartitionSubsystem.cpp:1185` | Policy 駆動ドライバ |
| `UWorldPartitionStreamingPolicy::UpdateStreamingState` | `WorldPartitionStreamingPolicy.cpp:273` | ハッシュ差分・非同期ディスパッチ |
| `UWorldPartitionStreamingPolicy::UpdateStreamingStateInternal` | `WorldPartitionStreamingPolicy.cpp:392` | 実計算（GameThread / WorkerThread 両対応） |
| `UWorldPartitionRuntimeHashSet::GetStreamingCells` | `RuntimeHashSet/WorldPartitionRuntimeHashSet.cpp` | R-tree によるセル検索 |
| `UWorldPartition::OnCellShown` / `OnCellHidden` | `WorldPartition.cpp:1974` / `1988` | HLOD・Policy へ通知 |
| `UWorldPartitionRuntimeCell::SetStreamingStatus` | `WorldPartitionRuntimeCell.cpp` | LevelStreaming へフォワード |
