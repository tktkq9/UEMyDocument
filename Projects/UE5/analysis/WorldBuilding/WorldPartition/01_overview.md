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
