# WorldBuilding ドキュメント チェックリスト

## 概要
- [x] `01_worldbuilding_overview.md` … WorldBuilding 全体概要
- [x] `_source_map.md` … ソースマップ

---

## サブフォルダ構成

### WorldPartition — ワールドパーティション
- [x] `WorldPartition/01_overview.md` … WorldPartition 概要
- [x] `WorldPartition/Details/a_spatial_hash.md` … UWorldPartitionRuntimeSpatialHash・セル分割・グリッド設定
- [x] `WorldPartition/Details/b_streaming_policy.md` … AWorldPartitionStreamingPolicy・StreamingSource・ロード/アンロード
- [x] `WorldPartition/Details/c_actor_desc.md` … FWorldPartitionActorDesc・ActorDescView・エディタ連携
- [x] `WorldPartition/Details/d_runtime_cell.md` … UWorldPartitionRuntimeCell・CellBounds・ContentBundle
- [x] `WorldPartition/Reference/ref_world_partition_api.md` … UWorldPartition API
- [x] `WorldPartition/Reference/ref_spatial_hash_api.md` … UWorldPartitionRuntimeSpatialHash API

### HLOD — 階層 LOD
- [x] `HLOD/01_overview.md` … HLOD 概要
- [x] `HLOD/Details/a_hlod_generation.md` … HLOD ビルド・ALODActor 生成・MeshMerge/Imposter
- [x] `HLOD/Details/b_hlod_layer.md` … UHLODLayer 設定・LOD 切り替え距離・パフォーマンス
- [x] `HLOD/Details/c_hlod_runtime.md` … ランタイム HLOD 切り替え・ストリーミング連携
- [x] `HLOD/Reference/ref_hlod_api.md` … UWorldPartitionHLOD / ALODActor API

### DataLayer — データレイヤー
- [x] `DataLayer/01_overview.md` … DataLayer 概要
- [x] `DataLayer/Details/a_data_layer_asset.md` … UDataLayerAsset・DataLayerType（Runtime/Editor）
- [x] `DataLayer/Details/b_runtime_toggle.md` … SetDataLayerRuntimeState・Loaded/Unloaded/Activated
- [x] `DataLayer/Details/c_editor_integration.md` … DataLayer エディタ UI・WorldPartition 連携
- [x] `DataLayer/Reference/ref_data_layer_api.md` … UDataLayerAsset / UDataLayerSubsystem API

### LevelStreaming — レベルストリーミング
- [x] `LevelStreaming/01_overview.md` … LevelStreaming 概要
- [x] `LevelStreaming/Details/a_level_streaming.md` … ULevelStreaming・RequestLevel・非同期ロード・LoadedLevel
- [x] `LevelStreaming/Details/b_seamless_travel.md` … Seamless/Non-Seamless Travel・TransitionMap・PersistentLevel
- [x] `LevelStreaming/Details/c_dynamic_streaming.md` … ULevelStreamingDynamic・ランタイム生成・Transform
- [x] `LevelStreaming/Reference/ref_streaming_api.md` … ULevelStreaming / ULevelStreamingDynamic API

### PCG — プロシージャルコンテンツ生成
- [x] `PCG/01_overview.md` … PCG 概要
- [x] `PCG/Details/a_pcg_graph.md` … UPCGGraph・FPCGGraphCompiler・ノード評価順序
- [x] `PCG/Details/b_pcg_elements.md` … UPCGSettings 派生・PointSampler/SurfaceSampler/MeshSampler
- [x] `PCG/Details/c_pcg_data.md` … FPCGPoint・FPCGPointData・FPCGSpatialData・Metadata
- [x] `PCG/Details/d_custom_nodes.md` … カスタム PCG ノード実装・C++/BP 拡張
- [x] `PCG/Reference/ref_pcg_api.md` … UPCGComponent / UPCGGraph API
- [x] `PCG/Reference/ref_pcg_elements.md` … 標準 PCG ノード全一覧

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| WorldPartition | 1/1 | 4/4 | 2/2 | 7/7 |
| HLOD | 1/1 | 3/3 | 1/1 | 5/5 |
| DataLayer | 1/1 | 3/3 | 1/1 | 5/5 |
| LevelStreaming | 1/1 | 3/3 | 1/1 | 5/5 |
| PCG | 1/1 | 4/4 | 2/2 | 7/7 |

**合計**: 概要 1 + ソースマップ 1 + サブフォルダ概要 5 + Details 17 + Reference 7 = **31 ファイル（全完了）**

---

## Ph4 — コード実行フロー追記（2026-04-19 完了）

- [x] `01_worldbuilding_overview.md`
- [x] `WorldPartition/01_overview.md`
- [x] `WorldPartition/Details/a_spatial_hash.md`
- [x] `WorldPartition/Details/b_streaming_policy.md`
- [x] `WorldPartition/Details/c_actor_desc.md`
- [x] `WorldPartition/Details/d_runtime_cell.md`
- [x] `HLOD/01_overview.md`
- [x] `HLOD/Details/a_hlod_generation.md`
- [x] `HLOD/Details/b_hlod_layer.md`
- [x] `HLOD/Details/c_hlod_runtime.md`
- [x] `DataLayer/01_overview.md`
- [x] `DataLayer/Details/a_data_layer_asset.md`
- [x] `DataLayer/Details/b_runtime_toggle.md`
- [x] `DataLayer/Details/c_editor_integration.md`
- [x] `LevelStreaming/01_overview.md`
- [x] `LevelStreaming/Details/a_level_streaming.md`
- [x] `LevelStreaming/Details/b_seamless_travel.md`
- [x] `LevelStreaming/Details/c_dynamic_streaming.md`
- [x] `PCG/01_overview.md`
- [x] `PCG/Details/a_pcg_graph.md`
- [x] `PCG/Details/b_pcg_elements.md`
- [x] `PCG/Details/c_pcg_data.md`
- [x] `PCG/Details/d_custom_nodes.md`

**Ph4 対象**: 23 ファイル（overview 1 + sub-overview 5 + Details 17）
