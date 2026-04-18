# Network システム全体概要

- 取得対象: `Engine/Source/Runtime/Engine/Classes/Engine/`, `Engine/Source/Runtime/Online/`
- 上位: [[UE5 解析インデックス]]

---

## UE5 ネットワークシステムの構成

UE5 のネットワークは **Authority（サーバー）/ Simulated（クライアント）** モデル。  
アクタープロパティは自動複製、関数は RPC で通信する。

| 概念 | クラス | 説明 |
|------|--------|------|
| ネットドライバー | `UIpNetDriver` | UDP ソケット管理 |
| 接続 | `UNetConnection` | クライアント接続 1 本 |
| チャンネル | `UActorChannel` | アクター複製チャンネル |
| レプリケーション | `UPROPERTY(Replicated)` | プロパティの自動同期 |
| RPC | `UFUNCTION(Server/Client/NetMulticast)` | 関数呼び出しの送受信 |
| GameState | `AGameStateBase` | 全クライアントへの複製状態 |
| PlayerState | `APlayerState` | プレイヤーごとの複製状態 |
| 予測 | `FNetworkPredictionData_Client` | クライアント側予測 |

---

## レプリケーションフロー

```
サーバー: Actor の UPROPERTY が変化
  └─ UNetDriver::ServerReplicateActors()
      └─ UActorChannel::ReplicateActor()
          ├─ FObjectReplicator::ReplicateProperties()  // 差分検出
          └─ UNetConnection::SendBunch()               // UDP 送信
              ↓
クライアント: UActorChannel::ProcessBunch()
  └─ FObjectReplicator::ReceiveProperties()
      └─ Actor::OnRep_*() コールバック
```

---

## 主要ソースファイル

| ファイル | 役割 |
|---------|------|
| `NetDriver.h/.cpp` | ネットワークドライバー基底 |
| `NetConnection.h/.cpp` | 接続管理 |
| `ActorChannel.h/.cpp` | アクター複製チャンネル |
| `ObjectReplicator.h/.cpp` | プロパティ複製エンジン |
| `NetworkPrediction*.h/.cpp` | クライアント予測 |
| `GameStateBase.h/.cpp` | GameState 複製 |
| `PlayerState.h/.cpp` | PlayerState 複製 |

---

## サブシステムドキュメント

| ドキュメント | 内容 |
|------------|------|
| `Details/a_replication.md` | UPROPERTY Replicated・RepNotify・条件複製 |
| `Details/b_rpc.md` | Server/Client/NetMulticast RPC・Reliable vs Unreliable |
| `Details/c_netdriver.md` | NetDriver・NetConnection・Bandwidth 管理 |
| `Details/d_prediction.md` | CharacterMovement 予測・FNetworkPredictionData |
| `Details/e_gamestate_playerstate.md` | GameState/PlayerState 複製パターン |
