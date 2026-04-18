# WorldBuilding システム全体概要

- 取得対象: `Engine/Source/Runtime/Engine/Classes/Engine/`, `Engine/Plugins/PCG/`
- 上位: [[UE5 解析インデックス]]

---

## UE5 WorldBuilding システムの構成

UE5 の大規模ワールド構築は **World Partition** が中心。  
ストリーミング・HLOD・DataLayer・PCG を組み合わせる。

| 機能 | クラス | 説明 |
|------|--------|------|
| World Partition | `UWorldPartition` | ワールドのグリッド分割・自動ストリーミング |
| Level Streaming | `ULevelStreaming` | 旧来のレベル非同期ロード |
| HLOD | `ALODActor` / `UWorldPartitionHLOD` | 遠距離の低解像度メッシュ |
| DataLayer | `UDataLayerAsset` | レイヤー別の表示切り替え |
| PCG | `UPCGComponent` | プロシージャルコンテンツ生成グラフ |
| World Composition | `UWorldComposition` | 旧来の World Composition（4.x 互換）|

---

## World Partition ストリーミングフロー

```
AWorldPartitionStreamingPolicy::Tick()
  └─ UWorldPartitionRuntimeSpatialHash::UpdateStreamingSources()
      ├─ カメラ位置 → ロードすべきセルを決定
      ├─ 未ロードセル → FWorldPartitionActorDescView をキュー
      └─ ULevelStreamingDynamic::RequestLevel()
          └─ 非同期ロード → ActorStreaming 完了後にワールドに追加
```

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `WorldPartition.h/.cpp` | World Partition 管理 |
| `WorldPartitionRuntimeSpatialHash.h/.cpp` | 空間ハッシュ・セル管理 |
| `WorldPartitionHLOD.h/.cpp` | HLOD 生成・管理 |
| `DataLayerAsset.h/.cpp` | DataLayer 定義 |
| `LevelStreaming.h/.cpp` | レベルストリーミング |
| `PCGComponent.h/.cpp` | PCG コンポーネント |
| `PCGGraph.h/.cpp` | PCG グラフ評価 |

---

## サブシステムドキュメント

| ドキュメント | 内容 |
|------------|------|
| `Details/a_world_partition.md` | World Partition・セル・ストリーミングポリシー |
| `Details/b_hlod.md` | HLOD 生成・WorldPartitionHLOD・LOD切り替え |
| `Details/c_data_layer.md` | DataLayer・ランタイム切り替え・エディタ連携 |
| `Details/d_level_streaming.md` | LevelStreaming・サブレベル・Seamless Travel |
| `Details/e_pcg.md` | PCG グラフ・ノード・Point/Mesh/Surface 生成 |
