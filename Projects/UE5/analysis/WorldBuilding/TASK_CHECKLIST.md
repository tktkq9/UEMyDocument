# WorldBuilding ドキュメント チェックリスト

## 概要
- [x] `01_worldbuilding_overview.md` … WorldBuilding 全体概要
- [x] `_source_map.md` … ソースマップ

---

## サブフォルダ構成

### WorldPartition — ワールドパーティション
- [x] `WorldPartition/01_overview.md` … WorldPartition 概要
- [ ] `WorldPartition/Details/a_spatial_hash.md` … UWorldPartitionRuntimeSpatialHash・セル分割・グリッド設定
- [ ] `WorldPartition/Details/b_streaming_policy.md` … AWorldPartitionStreamingPolicy・StreamingSource・ロード/アンロード
- [ ] `WorldPartition/Details/c_actor_desc.md` … FWorldPartitionActorDesc・ActorDescView・エディタ連携
- [ ] `WorldPartition/Details/d_runtime_cell.md` … UWorldPartitionRuntimeCell・CellBounds・ContentBundle
- [ ] `WorldPartition/Reference/ref_world_partition_api.md` … UWorldPartition API
- [ ] `WorldPartition/Reference/ref_spatial_hash_api.md` … UWorldPartitionRuntimeSpatialHash API

### HLOD — 階層 LOD
- [x] `HLOD/01_overview.md` … HLOD 概要
- [ ] `HLOD/Details/a_hlod_generation.md` … HLOD ビルド・ALODActor 生成・MeshMerge/Imposter
- [ ] `HLOD/Details/b_hlod_layer.md` … UHLODLayer 設定・LOD 切り替え距離・パフォーマンス
- [ ] `HLOD/Details/c_hlod_runtime.md` … ランタイム HLOD 切り替え・ストリーミング連携
- [ ] `HLOD/Reference/ref_hlod_api.md` … UWorldPartitionHLOD / ALODActor API

### DataLayer — データレイヤー
- [x] `DataLayer/01_overview.md` … DataLayer 概要
- [ ] `DataLayer/Details/a_data_layer_asset.md` … UDataLayerAsset・DataLayerType（Runtime/Editor）
- [ ] `DataLayer/Details/b_runtime_toggle.md` … SetDataLayerRuntimeState・Loaded/Unloaded/Activated
- [ ] `DataLayer/Details/c_editor_integration.md` … DataLayer エディタ UI・WorldPartition 連携
- [ ] `DataLayer/Reference/ref_data_layer_api.md` … UDataLayerAsset / UDataLayerSubsystem API

### LevelStreaming — レベルストリーミング
- [x] `LevelStreaming/01_overview.md` … LevelStreaming 概要
- [ ] `LevelStreaming/Details/a_level_streaming.md` … ULevelStreaming・RequestLevel・非同期ロード・LoadedLevel
- [ ] `LevelStreaming/Details/b_seamless_travel.md` … Seamless/Non-Seamless Travel・TransitionMap・PersistentLevel
- [ ] `LevelStreaming/Details/c_dynamic_streaming.md` … ULevelStreamingDynamic・ランタイム生成・Transform
- [ ] `LevelStreaming/Reference/ref_streaming_api.md` … ULevelStreaming / ULevelStreamingDynamic API

### PCG — プロシージャルコンテンツ生成
- [x] `PCG/01_overview.md` … PCG 概要
- [ ] `PCG/Details/a_pcg_graph.md` … UPCGGraph・FPCGGraphCompiler・ノード評価順序
- [ ] `PCG/Details/b_pcg_elements.md` … UPCGSettings 派生・PointSampler/SurfaceSampler/MeshSampler
- [ ] `PCG/Details/c_pcg_data.md` … FPCGPoint・FPCGPointData・FPCGSpatialData・Metadata
- [ ] `PCG/Details/d_custom_nodes.md` … カスタム PCG ノード実装・C++/BP 拡張
- [ ] `PCG/Reference/ref_pcg_api.md` … UPCGComponent / UPCGGraph API
- [ ] `PCG/Reference/ref_pcg_elements.md` … 標準 PCG ノード全一覧

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| WorldPartition | 0/1 | 0/4 | 0/2 | 0/7 |
| HLOD | 0/1 | 0/3 | 0/1 | 0/5 |
| DataLayer | 0/1 | 0/3 | 0/1 | 0/5 |
| LevelStreaming | 0/1 | 0/3 | 0/1 | 0/5 |
| PCG | 0/1 | 0/4 | 0/2 | 0/7 |

**合計**: 概要 1 + ソースマップ 1 + サブフォルダ概要 5 + Details 17 + Reference 7 = **31 ファイル**
