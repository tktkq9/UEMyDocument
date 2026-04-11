# D3D12 コマンド実行層

- 対象: `Runtime/D3D12RHI/Private/D3D12CommandContext.h/.cpp`, `D3D12CommandList.h/.cpp`, `D3D12Submission.h/.cpp`
- 関連Reference: [[ref_rhi_d3d12_context]]

---

## 概要

コマンド実行層は **FD3D12ContextCommon → FD3D12CommandList → FD3D12Queue** の流れで  
RHI のドローコールを D3D12 API (`ID3D12GraphicsCommandList`) に変換して GPU に投入する。

---

## クラス構成

```
IRHICommandContext                     ← RHI 抽象インターフェース
  └── FD3D12ContextCommon              ← D3D12 実装の中心クラス
        ├── FD3D12CommandList*         ← 現在記録中のコマンドリスト
        ├── FD3D12Device*              ← 親デバイス参照
        ├── FD3D12Queue*               ← 送信先キュー
        ├── FD3D12StateCachePrivate    ← ステートキャッシュ（冗長設定省略）
        └── FD3D12ExplicitDescriptorCache ← ディスクリプタコピーキャッシュ

FD3D12CommandList                      ← ID3D12GraphicsCommandList ラッパー
  ├── ID3D12GraphicsCommandList*       ← D3D12 ネイティブコマンドリスト
  └── FD3D12CommandAllocator*          ← コマンドアロケータ（再利用プール管理）

FD3D12Submission                       ← コマンドリスト投入管理（フレーム単位）
  └── FD3D12Payload                    ← 1フレームのコマンドバッチ
```

---

## FD3D12ContextCommon

`IRHICommandContext` の D3D12 実装。`RHIXxx()` 仮想関数を D3D12 API 呼び出しに変換する。

```cpp
class FD3D12ContextCommon : public IRHICommandContext
{
public:
    // ---- IRHICommandContext 仮想関数の実装 ----

    virtual void RHISetGraphicsPipelineState(
        FRHIGraphicsPipelineState* GraphicsState,
        uint32 StencilRef,
        bool bApplyAdditionalState) override;
    // → CommandListHandle->SetPipelineState(D3D12PSO)

    virtual void RHIDrawIndexedPrimitive(
        FRHIBuffer* IndexBuffer, int32 BaseVertexIndex,
        uint32 FirstInstance, uint32 NumVertices,
        uint32 StartIndex, uint32 NumPrimitives, uint32 NumInstances) override;
    // → CommandListHandle->DrawIndexedInstanced(...)

    virtual void RHIDispatchComputeShader(
        uint32 X, uint32 Y, uint32 Z) override;
    // → CommandListHandle->Dispatch(X, Y, Z)

    virtual void RHICopyTexture(
        FRHITexture* SourceTexture, FRHITexture* DestTexture,
        const FRHICopyTextureInfo& CopyInfo) override;
    // → CommandListHandle->CopyTextureRegion(...)

    // ---- ステート管理 ----

    // コマンドリストを新規取得して記録開始
    void OpenCommandList();

    // コマンドリストをクローズして投入キューへ
    void CloseCommandList();

    // 保留中のリソースバリアをまとめてフラッシュ
    void FlushResourceBarriers();

protected:
    FD3D12CommandList* CommandListHandle;      // 現在の記録先
    FD3D12Device*      ParentDevice;
    FD3D12Queue*       Queue;
    FD3D12StateCachePrivate StateCache;        // ステートキャッシュ
};
```

---

## FD3D12CommandList

`ID3D12GraphicsCommandList` の薄いラッパー。記録・クローズ・再利用を管理する。

```cpp
class FD3D12CommandList
{
public:
    // コマンド記録開始（アロケータをリセットして記録可能状態に）
    void Open(FD3D12CommandAllocator* Allocator, ID3D12PipelineState* InitialPSO = nullptr);

    // 記録終了（Close 後は実行可能になる）
    void Close();

    // D3D12 ネイティブオブジェクトへのアクセス
    ID3D12GraphicsCommandList*  GetCommandList()     const;
    ID3D12GraphicsCommandList2* GetCommandList2()    const;  // WriteBuf 系
    ID3D12GraphicsCommandList4* GetCommandList4()    const;  // DXR 対応

    // フェンス値（GPU 完了追跡用）
    uint64 GetFenceValue() const;

    // アロケータへのアクセス
    FD3D12CommandAllocator* GetCurrentAllocator() const;
};
```

---

## FD3D12CommandAllocator

```cpp
class FD3D12CommandAllocator
{
public:
    // GPU 完了後に Reset して再利用
    void Reset();

    bool IsReady() const;  // GPU がこのアロケータのコマンドを実行し終えたか

    ID3D12CommandAllocator* GetCommandAllocator() const;
};
```

アロケータはコマンドリストが**実行中は変更不可**なため、プール管理される。  
フェンス完了を確認してから `Reset()` を呼ぶことで再利用できる。

---

## FD3D12Submission（コマンド投入管理）

```cpp
// 1フレーム分の投入ペイロード
struct FD3D12Payload
{
    TArray<FD3D12CommandList*> CommandLists;    // 実行するコマンドリスト群
    TArray<FD3D12FenceEventPair> FenceEvents;   // 待機フェンス
    TArray<FD3D12FenceEventPair> SignalFences;  // シグナルフェンス
};

// 投入スレッド
class FD3D12SubmissionThread
{
    // Payload をデキューして ID3D12CommandQueue::ExecuteCommandLists を呼ぶ
    void ProcessPayload(FD3D12Payload& Payload);
};
```

---

## コマンド記録・投入フロー

```
[Render Thread → RHI Thread]
FRHICommandListImmediate::DrawIndexedPrimitive(...)
    ↓ コマンドを FRHICmdList に積む（EnqueueCommand）

[RHI Thread]
FD3D12ContextCommon::RHIDrawIndexedPrimitive(...)
    │
    ├─ FlushResourceBarriers()            ← 保留バリアを先にまとめて発行
    │     CommandListHandle->ResourceBarrier(N バリアをまとめて)
    │
    └─ CommandListHandle->DrawIndexedInstanced(...)
         ↑ FD3D12CommandList::GetCommandList() の ID3D12GraphicsCommandList

[フレーム終了時]
FD3D12ContextCommon::CloseCommandList()
    └─ CommandListHandle->Close()

FD3D12Queue::ExecuteCommandList(CommandListHandle)
    └─ FD3D12SubmissionThread に Payload を Enqueue

[Submission Thread]
ID3D12CommandQueue::ExecuteCommandLists(...)  ← GPU へ投入
ID3D12CommandQueue::Signal(Fence, FenceValue) ← 完了フェンスをシグナル
```

---

## ステートキャッシュ（FD3D12StateCachePrivate）

D3D12 では `SetGraphicsRootDescriptorTable` 等の API は**冗長呼び出しでも問題ないが非効率**。  
`FD3D12StateCachePrivate` が現在の設定値をキャッシュし、変化がある場合のみ D3D12 API を呼ぶ。

```cpp
class FD3D12StateCachePrivate
{
    // 現在の PSO / ルートシグネチャ / 頂点バッファ / ビューポート 等をキャッシュ
    FD3D12PipelineState* CurrentPipelineState;
    D3D12_VIEWPORT       CurrentViewport;
    D3D12_RECT           CurrentScissorRect;
    // …

    // 変化があれば実際に SetXxx を呼ぶ
    void ApplyState(FD3D12ContextCommon& Context);
};
```

---

## リソースバリアの処理

```cpp
// バリアを一時バッファに積んでおく（FD3D12ContextCommon 内）
void AddPendingResourceBarrier(
    FD3D12Resource* Resource,
    D3D12_RESOURCE_STATES Before,
    D3D12_RESOURCE_STATES After);

// まとめて発行（ドローコールの直前 or 明示 Flush 時）
void FlushResourceBarriers()
{
    // TArray<D3D12_RESOURCE_BARRIER> をまとめて ResourceBarrier() に渡す
    CommandList->ResourceBarrier(PendingBarriers.Num(), PendingBarriers.GetData());
    PendingBarriers.Reset();
}
```

RDG が自動生成した `ERHIAccess` トランジションはここで D3D12 バリアに変換される（[[e_rhi_rdg_bridge]] 参照）。

---

## コード実行フロー

```
[コマンドリスト記録サイクル]

FD3D12ContextCommon::OpenCommandList()    D3D12CommandContext.cpp:351
  ├─ CommandAllocatorManager からアロケータを取得（または新規生成）
  ├─ ID3D12Device::CreateCommandList() or Reset()
  └─ StateCache.ClearState()             ← キャッシュを初期化

コマンド記録ループ:
  FD3D12CommandContext::RHISetGraphicsPipelineState()  D3D12Commands.cpp:390
    └─ StateCache に PSO を記録（冗長変更はスキップ）
  
  FD3D12CommandContext::RHIDrawIndexedPrimitive()      D3D12Commands.cpp:1270
    ├─ StateCache.ApplyState()           ← 変化分のみ SetXxx を呼ぶ
    ├─ FlushResourceBarriers()           D3D12CommandList.cpp:70
    │    └─ ID3D12GraphicsCommandList::ResourceBarrier(PendingBarriers)
    └─ ID3D12GraphicsCommandList::DrawIndexedInstanced()

  FD3D12CommandContext::RHIDispatchComputeShader()     D3D12Commands.cpp:140
    ├─ SetupDispatch()                   D3D12Commands.cpp:97
    └─ ID3D12GraphicsCommandList::Dispatch()

FD3D12ContextCommon::CloseCommandList()  D3D12CommandContext.cpp:378
  ├─ FlushResourceBarriers()             ← 残りバリアを発行
  └─ ID3D12GraphicsCommandList::Close()

[GPU 投入]
FD3D12Submission::ProcessPayload()       D3D12Submission.cpp:553
  └─ ID3D12CommandQueue::ExecuteCommandLists()  D3D12Submission.cpp:827
       └─ ID3D12CommandQueue::Signal()   ← フェンス値をインクリメント

[アロケータ再利用]
GetLastCompletedFence() >= AllocatorFenceValue
  └─ FD3D12CommandAllocator::Reset()   → FreeAllocators プールへ返却
```

### 関与クラス・関数一覧

| クラス / 関数 | ファイル:行 | 役割 |
|-------------|-----------|------|
| `FD3D12ContextCommon::OpenCommandList()` | `D3D12CommandContext.cpp:351` | アロケータ取得・コマンドリスト Open |
| `FD3D12ContextCommon::CloseCommandList()` | `D3D12CommandContext.cpp:378` | バリアフラッシュ後 Close |
| `FD3D12ContextCommon::FlushResourceBarriers()` | `D3D12CommandList.cpp:70` | 保留バリアを一括発行 |
| `FD3D12CommandContext::RHISetGraphicsPipelineState()` | `D3D12Commands.cpp:390` | StateCache 経由で PSO を設定 |
| `FD3D12CommandContext::RHIDrawIndexedPrimitive()` | `D3D12Commands.cpp:1270` | DrawIndexedInstanced に変換 |
| `FD3D12CommandContext::RHIDispatchComputeShader()` | `D3D12Commands.cpp:140` | Dispatch に変換 |
| `FD3D12CommandContext::SetupDispatch()` | `D3D12Commands.cpp:97` | Compute 前の共通セットアップ |
| `FD3D12StateCachePrivate::ApplyState()` | `D3D12StateCachePrivate.h` | 差分のみ D3D12 API を呼ぶ |
| `ID3D12CommandQueue::ExecuteCommandLists()` | `D3D12Submission.cpp:827` | GPU への実際の投入 |

---

## 関連リファレンス

- [[ref_rhi_d3d12_context]] … FD3D12ContextCommon / FD3D12CommandList / FD3D12CommandAllocator の詳細
