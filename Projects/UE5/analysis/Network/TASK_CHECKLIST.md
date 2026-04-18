# Network ドキュメント チェックリスト

## 概要
- [ ] `01_network_overview.md` … Network 全体概要（更新予定）
- [ ] `_source_map.md` … ソースマップ

---

## サブフォルダ構成

### Replication — プロパティ複製
- [ ] `Replication/01_overview.md` … Replication 概要
- [ ] `Replication/Details/a_property_replication.md` … UPROPERTY(Replicated)・RepNotify・DOREPLIFETIME マクロ
- [ ] `Replication/Details/b_rep_layout.md` … FRepLayout・FRepState・差分検出・シリアライズ
- [ ] `Replication/Details/c_conditional_rep.md` … COND_* 条件複製・FDoRepLifetimeParams・Push Model
- [ ] `Replication/Details/d_actor_replication.md` … AActor 複製ライフサイクル・Relevancy・Priority
- [ ] `Replication/Reference/ref_replication_api.md` … GetLifetimeReplicatedProps / DOREPLIFETIME API
- [ ] `Replication/Reference/ref_rep_macros.md` … レプリケーション関連マクロ一覧

### RPC — リモートプロシージャコール
- [ ] `RPC/01_overview.md` … RPC 概要
- [ ] `RPC/Details/a_rpc_types.md` … Server/Client/NetMulticast・Reliable/Unreliable
- [ ] `RPC/Details/b_rpc_validation.md` … WithValidation・ServerRPC バリデーション・権限チェック
- [ ] `RPC/Details/c_rpc_serialization.md` … パラメータシリアライズ・FNetBitWriter・圧縮
- [ ] `RPC/Reference/ref_rpc_api.md` … RPC 宣言マクロ・UFUNCTION ネットワーク指定子

### NetDriver — 接続管理・チャンネル
- [ ] `NetDriver/01_overview.md` … NetDriver 概要
- [ ] `NetDriver/Details/a_net_driver.md` … UNetDriver・InitListen/InitConnect・TickFlush
- [ ] `NetDriver/Details/b_net_connection.md` … UNetConnection・チャンネル管理・パケット処理
- [ ] `NetDriver/Details/c_channel.md` … UChannel・UActorChannel・UControlChannel・UVoiceChannel
- [ ] `NetDriver/Details/d_packet.md` … パケット構造・SequenceNumber・Ack・帯域制御
- [ ] `NetDriver/Reference/ref_netdriver_api.md` … UNetDriver / UNetConnection API

### Prediction — クライアント予測
- [ ] `Prediction/01_overview.md` … Prediction 概要
- [ ] `Prediction/Details/a_client_prediction.md` … クライアント予測の原理・ServerMove/ClientAck
- [ ] `Prediction/Details/b_correction.md` … サーバー補正・リコンシリエーション・スムージング
- [ ] `Prediction/Details/c_prediction_data.md` … FNetworkPredictionData_Client/Server・SavedMove
- [ ] `Prediction/Reference/ref_prediction_api.md` … FNetworkPredictionData API

### Iris — 新レプリケーションシステム
- [ ] `Iris/01_overview.md` … Iris 概要
- [ ] `Iris/Details/a_iris_architecture.md` … FReplicationSystem・FReplicationWriter/Reader
- [ ] `Iris/Details/b_iris_protocol.md` … FReplicationProtocol・NetObject・NetHandle
- [ ] `Iris/Details/c_iris_filtering.md` … FNetObjectFilter・Relevancy・Priority（Iris 版）
- [ ] `Iris/Reference/ref_iris_api.md` … UIrisNetDriver / FReplicationSystem API

### Replay — リプレイシステム
- [ ] `Replay/01_overview.md` … Replay 概要
- [ ] `Replay/Details/a_demo_netdriver.md` … UDemoNetDriver・録画/再生フロー・チェックポイント
- [ ] `Replay/Details/b_replay_streaming.md` … FQueuedReplayTask・LocalFile/HTTP ストリーミング
- [ ] `Replay/Reference/ref_replay_api.md` … UDemoNetDriver / UGameInstance リプレイ API

---

## 進捗サマリ

| サブフォルダ | 概要 | Details | Reference | 完了 |
|------------|------|---------|-----------|------|
| Replication | 0/1 | 0/4 | 0/2 | 0/7 |
| RPC | 0/1 | 0/3 | 0/1 | 0/5 |
| NetDriver | 0/1 | 0/4 | 0/1 | 0/6 |
| Prediction | 0/1 | 0/3 | 0/1 | 0/5 |
| Iris | 0/1 | 0/3 | 0/1 | 0/5 |
| Replay | 0/1 | 0/2 | 0/1 | 0/4 |

**合計**: 概要 1 + ソースマップ 1 + サブフォルダ概要 6 + Details 19 + Reference 7 = **34 ファイル**
