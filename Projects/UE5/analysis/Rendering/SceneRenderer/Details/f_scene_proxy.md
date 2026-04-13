# F: FPrimitiveSceneProxy — GameThread ↔ RenderThread 橋渡し

- 対象: `PrimitiveSceneProxy.h/.cpp`, `PrimitiveSceneInfo.h/.cpp`
- 上位: [[01_rendering_overview]]
- Reference: [[ref_scene]] | [[ref_view_info]]

---

## 概要

`FPrimitiveSceneProxy` は `UPrimitiveComponent`（GameThread 側）の情報を  
**RenderThread 安全な形で複製・保持する**ミラーオブジェクト。

```
GameThread                         RenderThread
─────────────────────────────────  ────────────────────────────────
UPrimitiveComponent                FPrimitiveSceneProxy    ← ここのドキュメント
  │  CreateSceneProxy()              │
  │  ──────────────────────────────▶│ 生成（RT 側にコピーされる）
  │                                  │
  └─ FPrimitiveSceneInfo             │  FPrimitiveSceneInfo
     （FScene 内の管理構造体）         └─ GetPrimitiveSceneInfo()
```

- `FPrimitiveSceneProxy`：描画ロジックとデータ（マテリアル・バウンド・フラグ）
- `FPrimitiveSceneInfo`：FScene 内部での管理情報（インデックス・キャッシュ）

---

## ライフサイクル

```
【登録】
UPrimitiveComponent::RegisterComponent()  [GameThread]
  └─ FScene::AddPrimitive(Proxy)
        ENQUEUE_RENDER_COMMAND
          └─ FPrimitiveSceneProxy::CreateRenderThreadResources(RHICmdList) [RenderThread]
             → GPU リソース（VB/IB/テクスチャ）の確保

【トランスフォーム更新】
UPrimitiveComponent::SendRenderTransform()  [GameThread]
  └─ ENQUEUE_RENDER_COMMAND
       └─ FPrimitiveSceneProxy::SetTransform(RHICmdList, LocalToWorld, Bounds, ...)

【登録解除】
UPrimitiveComponent::UnregisterComponent()  [GameThread]
  └─ FScene::RemovePrimitive(Proxy)
        ENQUEUE_RENDER_COMMAND
          └─ ~FPrimitiveSceneProxy()  [RenderThread]
```

---

## 主要な仮想関数

| 関数 | 説明 |
|------|------|
| `CreateRenderThreadResources(RHICmdList)` | GPU リソース確保（登録直後に RT で呼ばれる） |
| `GetDynamicMeshElements(Views, ViewFamily, VisibilityMap, Collector)` | 動的描画の `FMeshBatch` を Collector に追加 |
| `DrawStaticElements(PDI)` | 静的描画の `FStaticMeshBatch` を登録（初回のみ） |
| `GetViewRelevance(View)` | このプリミティブがどのパスに参加するかを `FPrimitiveViewRelevance` で返す |
| `GetTypeHash()` | ソート用ハッシュ（純粋仮想） |
| `GetMemoryFootprint()` | メモリ使用量デバッグ用（推奨実装） |

---

## 静的描画 vs 動的描画

| 種別 | 関数 | 特徴 |
|------|------|------|
| **静的（Static）** | `DrawStaticElements()` | 登録時に1回だけ `FStaticMeshBatch` を生成。MeshDrawCommand にキャッシュされる。毎フレームコストなし |
| **動的（Dynamic）** | `GetDynamicMeshElements()` | 毎フレーム `FMeshBatch` を生成。アニメーション・頂点シェーダーアニメに使用 |

`GetViewRelevance()` の `bStaticRelevance` / `bDynamicRelevance` でどちらのパスに入るかを決定。

---

## `FPrimitiveViewRelevance`（パス参加フラグ）

```cpp
struct FPrimitiveViewRelevance
{
    uint8 bDrawRelevance      : 1;  // 描画対象か（false なら全スキップ）
    uint8 bStaticRelevance    : 1;  // Static MeshDrawCommand パス
    uint8 bDynamicRelevance   : 1;  // Dynamic Mesh パス（毎フレーム生成）
    uint8 bOpaque             : 1;  // GBuffer / BasePass 不透明
    uint8 bTranslucentSurface : 1;  // 半透明パス
    uint8 bShadowRelevance    : 1;  // シャドウパス
    uint8 bVelocityRelevance  : 1;  // ベロシティバッファ書き込み
    uint8 bDecal              : 1;  // デカールパス
};
```

---

## 主要データメンバ（PrimitiveSceneProxy.h）

| メンバ | 型 | 説明 |
|--------|-----|------|
| `Bounds` | `FBoxSphereBounds` | ワールド空間バウンド（カリング用） |
| `LocalBounds` | `FBoxSphereBounds` | ローカル空間バウンド |
| `PrimitiveSceneInfo` | `FPrimitiveSceneInfo*` | FScene 内の管理構造体へのポインタ |
| `LocalToWorld` | `FMatrix` | ワールド変換行列 |
| `bCastShadow` | `bool` | シャドウキャスト有無 |
| `bCastShadowAsTwoSided` | `bool` | 両面シャドウ |
| `LightingChannels` | `FLightingChannels` | ライティングチャンネル |

---

## GameThread ↔ RenderThread 通信パターン

```cpp
// ゲームスレッドからレンダースレッドへプロパティを送る典型的な実装例
void UMyComponent::SetColor_GameThread(FLinearColor NewColor)
{
    FMySceneProxy* Proxy = (FMySceneProxy*)SceneProxy; // 生のポインタ参照は禁止
    ENQUEUE_RENDER_COMMAND(SetColorCommand)(
        [Proxy, NewColor](FRHICommandListImmediate& RHICmdList)
        {
            Proxy->SetColor_RenderThread(NewColor); // RenderThread で安全に書き換え
        });
}
```

**重要ルール：**
- GameThread から Proxy のメンバを直接読み書きしてはいけない
- `ENQUEUE_RENDER_COMMAND` でラムダを投入して RenderThread 上で操作する
- Proxy の生存期間は RenderThread が管理（`~FPrimitiveSceneProxy()` は RT で呼ばれる）

---

## コード実行フロー

```
UPrimitiveComponent::RegisterComponent()  [GameThread]
  │
  └─ UWorld::Scene->AddPrimitive(UPrimitiveComponent*)
       │
       ├─ UPrimitiveComponent::CreateSceneProxy()  ← サブクラスがオーバーライド
       │   → FPrimitiveSceneProxy* Proxy = new FMySceneProxy(this)
       │
       ├─ ENQUEUE_RENDER_COMMAND
       │     FPrimitiveSceneProxy::CreateRenderThreadResources(RHICmdList)
       │       → VB / IB / テクスチャなど GPU リソース確保
       │
       └─ FPrimitiveSceneInfo* Info = new FPrimitiveSceneInfo(Proxy, Scene)
              FScene::PrimitiveSceneProxies[] に追加
              FScene::PrimitiveBounds[] に Bounds 格納

【毎フレームのパス参加判定】
BeginInitViews()
  └─ ComputeAndMarkRelevanceForViewParallel()
       └─ Proxy->GetViewRelevance(View)  → FPrimitiveViewRelevance

GatherDynamicMeshElements()
  └─ Proxy->GetDynamicMeshElements(Views, ViewFamily, VisibilityMap, Collector)
       → FMeshBatch を Collector に追加 → DrawCall 発行
```

---

## 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|--------|------|
| `FPrimitiveSceneProxy` | `PrimitiveSceneProxy.h` | RT 側の Component ミラー（基底クラス） |
| `FPrimitiveSceneInfo` | `PrimitiveSceneInfo.h` | FScene 内の管理構造体 |
| `FPrimitiveViewRelevance` | `PrimitiveViewRelevance.h` | パス参加フラグセット |
| `FMeshBatch` | `MeshBatch.h` | 1 ドローコール単位のメッシュデータ |
| `FMeshElementCollector` | `SceneRendering.h` | GetDynamicMeshElements の出力先 |
| `UPrimitiveComponent::CreateSceneProxy()` | `PrimitiveComponent.h` | Proxy 生成エントリポイント（GT） |
| `FScene::AddPrimitive()` / `RemovePrimitive()` | `RendererScene.cpp` | FScene への登録・解除 |
