# C: RDG パス実行モデル

- 対象: `RenderGraphPass.h`, `RenderGraphDefinitions.h`
- 上位: [[10_rdg_overview]]
- Reference: [[ref_rdg_pass]]

---

## ERDGPassFlags — パス種別フラグ

```cpp
enum class ERDGPassFlags : uint16 {
    None          = 0,
    Raster        = 1 << 0,  // グラフィックスパイプライン（描画コマンド）
    Compute       = 1 << 1,  // グラフィックスパイプライン上のコンピュート
    AsyncCompute  = 1 << 2,  // 非同期コンピュートパイプライン
    Copy          = 1 << 3,  // コピーコマンド（CopyTexture 等）
    NeverCull     = 1 << 4,  // カリング禁止（副作用のあるパスに付ける）
    SkipRenderPass = 1 << 5, // RenderPassBegin/End をスキップ
    NeverMerge    = 1 << 6,  // RenderPass マージを禁止
    NeverParallel = 1 << 7,  // レンダースレッドで同期実行を強制
    Readback      = Copy | NeverCull,  // GPU → CPU 読み戻し
};
```

### フラグ選択ガイド

| やりたいこと | 使うフラグ |
|-------------|----------|
| Dispatch（ComputeShader） | `Compute` |
| 非同期 Dispatch | `AsyncCompute` |
| DrawCall | `Raster` |
| テクスチャコピー | `Copy` |
| GPU → CPU 読み戻し | `Readback` |
| 常に実行（外部出力あり） | `NeverCull` を追加 |
| 既存 RHI コードを組み込む | `NeverCull \| SkipRenderPass` |

---

## ERDGPassTaskMode — 実行モード

```cpp
enum class ERDGPassTaskMode : uint8 {
    Inline,  // レンダースレッドで同期実行（デフォルト）
    Await,   // Task グラフで実行、Execute() 内で待機
    Async,   // Task グラフで実行、手動待機必須
};
```

ラムダの引数型がモードを決定する:

| ラムダ引数 | モード |
|-----------|--------|
| `FRHICommandListImmediate&` | Inline |
| `FRHICommandList&` | Await |
| `FRDGAsyncTask, FRHICommandList&` | Async |

---

## FRDGPass — パスの内部構造

```cpp
class FRDGPass
{
public:
    const TCHAR*       GetName()     const;
    ERDGPassFlags      GetFlags()    const;
    ERHIPipeline       GetPipeline() const;   // Graphics / AsyncCompute
    FRDGPassHandle     GetHandle()   const;
    uint32             GetWorkload() const;   // 相対負荷（並列スケジューリング用）
    ERDGPassTaskMode   GetTaskMode() const;

    bool IsParallelExecuteAllowed() const;
    bool IsMergedRenderPassBegin()  const;  // マージされた RenderPass の開始パス
    bool IsMergedRenderPassEnd()    const;  // マージされた RenderPass の終端パス
    bool IsAsyncCompute()          const;
    bool IsCulled()                const;   // 参照なしでスキップ

protected:
    FRDGPass(FRDGEventName&&, FRDGParameterStruct, ERDGPassFlags, ERDGPassTaskMode);
};
```

### 派生クラス

```cpp
// ラムダ付きパス（AddPass で生成される内部型）
template <typename ParameterStructType, typename ExecuteLambdaType>
class TRDGLambdaPass : public FRDGPass { ... };

// 並列コマンドリスト版（AddDispatchPass で生成）
class FRDGDispatchPass : public FRDGPass { ... };

// センチネル（グラフの先頭・末尾マーカー）
class FRDGSentinelPass : public FRDGPass { ... };
```

---

## RenderPass マージ

連続する `ERDGPassFlags::Raster` パスが同じレンダーターゲットを使う場合、  
RDG は自動的にそれらを 1 つの RenderPass にマージしてバリアを削減する。

```
[マージ前]           [マージ後]
Pass A (Raster)  →  BeginRenderPass
BeginRenderPass       Pass A (DrawCall)
EndRenderPass         Pass B (DrawCall)
Pass B (Raster)       EndRenderPass
BeginRenderPass
EndRenderPass
```

マージを禁止する場合は `NeverMerge` を追加する:

```cpp
GraphBuilder.AddPass(..., ERDGPassFlags::Raster | ERDGPassFlags::NeverMerge, ...);
```

---

## AsyncCompute（非同期コンピュート）

```cpp
// AsyncCompute パス（グラフィックスパイプラインと並走）
GraphBuilder.AddPass(
    RDG_EVENT_NAME("AsyncComputePass"),
    PassParams,
    ERDGPassFlags::AsyncCompute,
    [](FRHIComputeCommandList& RHICmdList)
    {
        // グラフィックスパスと並走して実行される
        RHICmdList.DispatchComputeShader(...);
    });
```

AsyncCompute の使用条件:
- `GRHISupportsAsyncCompute` が true のプラットフォーム（コンソール・DX12）
- 依存関係がない（書き込み先のリソースがグラフィックスパスで読まれない）

---

## パスカリングと NeverCull

RDG はグラフを解析し、どのパスも参照しない出力を持つパスを自動的にカリングする。  
画面に表示されない（最終出力に繋がらない）パスは自動除外される。

**カリングされないようにする場合:**

```cpp
// 外部副作用のあるパス（GPU メモリ書き出し・バッファ読み戻し等）
GraphBuilder.AddPass(
    RDG_EVENT_NAME("ReadbackPass"),
    PassParams,
    ERDGPassFlags::Readback,  // Readback = Copy | NeverCull
    [](FRHICommandList& RHICmdList)
    {
        // GPU → CPU へのバッファコピー
    });
```

---

## パス依存性の明示

```cpp
// パス間の依存を明示（AsyncCompute の同期点制御に使用）
GraphBuilder.AddPassDependency(ProducerPass, ConsumerPass);

// パスの相対負荷を設定（並列スケジューラのヒント）
// 1 がデフォルト。重い DrawCall を多く含む場合は大きい値に
GraphBuilder.SetPassWorkload(MyPass, 4);
```

---

## グラフコンパイルの全体像

```
Execute() 呼び出し時:

1. SetupPassDependencies()
   ← 各パスのパラメータを走査してリソース参照グラフを構築

2. CullPasses()
   ← 出力が誰にも使われないパスを除外（DFS で後ろから解析）

3. CollectPassBarriers()
   ← リソースアクセスパターンを解析して RHITransitionInfo を生成

4. MergeRenderPasses()
   ← 連続 Raster パスを BeginRenderPass/EndRenderPass でくくる

5. DetermineAsyncComputeRegions()
   ← AsyncCompute パスのフェンス挿入位置を決定

6. ExecutePasses()
   ← パスを順番に実行（並列モード時は Task グラフ経由）
```

---

## 関連リファレンス

| リファレンス | 対象ソース |
|------------|----------|
| [[ref_rdg_pass]] | `RenderGraphPass.h/.cpp` |
| [[ref_rdg_private]] | `RenderGraphPrivate.cpp` |

---

## コード実行フロー

### エントリポイント

```
GraphBuilder.AddPass(Name, Params, Flags, Lambda)
  │
  ├─ TRDGLambdaPass<Params, Lambda> を FRDGAllocator で確保
  ├─ ERDGPassTaskMode を Lambda の引数型から推論
  │   FRHICommandListImmediate& → Inline
  │   FRHICommandList&          → Await
  │   FRDGAsyncTask, ...        → Async
  │
  ├─ SetupParameterPass()         RenderGraphBuilder.cpp:2624
  │   └─ FRDGParameterStruct::Enumerate でリソース依存を収集
  │      IF_RDG_ENABLE_TRACE → Trace.AddTexturePassDependency / AddBufferPassDependency
  │
  ├─ IF_RDG_ENABLE_DEBUG → UserValidation.ValidateAddPass(Params, ...)
  │
  └─ FRDGPassRegistry に登録

──── Execute() 内 ────

Compile()
  ├─ SetupPassDependencies()      :2319
  │   └─ 全パスの FTextureState / FBufferState を走査
  │      FirstPass / LastPass / AccessModes を確定
  │
  ├─ CullPasses()
  │   └─ NeverCull / 外部抽出 Producer を Root に DFS
  │      カリングされたパスは FRDGPass::bCulled = true
  │
  ├─ MergeRenderPasses()
  │   └─ 同一 RT を持つ隣接 Raster パスを統合
  │      IsMergedRenderPassBegin() / IsMergedRenderPassEnd() フラグを設定
  │
  └─ SetupAsyncComputeRegions()
      └─ AsyncCompute パスの FRDGPass::AsyncComputeFence を設置

ExecutePasses()                   :2036
  └─ [パスごとのループ]
      ├─ ExecutePassPrologue()    :3395
      ├─ ExecutePass()            :3482
      │   ├─ Inline  → ExecuteSerialPass()  :3497（レンダースレッドで同期実行）
      │   ├─ Await   → タスクグラフに投入し完了まで待機
      │   └─ Async   → タスクグラフに投入・手動待機
      └─ ExecutePassEpilogue()    :3428
```

### フロー詳細

1. **AddPass → TRDGLambdaPass 生成**
   ```cpp
   // AddPass の内部（擬似コード）
   auto* Pass = Allocator.Alloc<TRDGLambdaPass<Params, Lambda>>(
       MoveTemp(Name), Params, Flags, TaskMode, MoveTemp(Lambda));
   Passes.Insert(Pass);  // FRDGPassRegistry に追加
   ```
   - ラムダはこの時点では **実行されない**（defer される）
   - `SetupParameterPass()` `:2624` でパラメータ構造体を走査してリソース参照を記録する

2. **SetupParameterPass() — `:2624`**
   ```
   FRDGParameterStruct::EnumerateTextures(Func)
     → UBMT_RDG_TEXTURE / SRV / UAV ごとに Func を呼び出す
     → FRDGPass::TextureStates に (Texture, AccessMode) を追加
   FRDGParameterStruct::EnumerateBuffers(Func)
     → FRDGPass::BufferStates に (Buffer, AccessMode) を追加
   ```
   - この情報が `SetupPassDependencies()` での First/Last パス解決に使われる

3. **CullPasses() — カリング判定**
   ```
   DFS（後ろから前に走査）:
     NeverCull フラグのパス          → Root（常に残す）
     QueueTextureExtraction のソース → Root
     Root に依存するパス             → Root に昇格

   bCulled = true のパスは ExecutePasses() でスキップされる
   ```

4. **MergeRenderPasses() — Raster パスのマージ**
   ```
   条件: 隣接する Raster パスが同一 FRenderTargetBinding を持つ
         かつ NeverMerge フラグなし
         かつ間にバリアが不要

   マージされると:
     先頭パス: IsMergedRenderPassBegin() == true  → BeginRenderPass を発行
     末尾パス: IsMergedRenderPassEnd() == true    → EndRenderPass を発行
     中間パス: どちらも false                    → BeginRenderPass/EndRenderPass を省略
   ```

5. **ExecutePass() — `:3482`**
   ```cpp
   ExecutePass(Pass):
     IF bCulled → スキップ
     ExecutePassPrologue(Pass)  :3395
       ├─ BarrierBatchBegin を Submit（RHI Transition Begin）
       ├─ Pass->GetPipeline() が AsyncCompute なら AsyncComputeFence を発行
       └─ BeginRenderPass（IsMergedRenderPassBegin の場合のみ）
     
     ExecuteSerialPass(Pass)  :3497
       ├─ Pass->Execute(RHICmdList) ← ユーザーのラムダを実行
       └─ IF_RDG_ENABLE_DEBUG → UserValidation.ValidateExecutePassEnd(Pass)
     
     ExecutePassEpilogue(Pass)  :3428
       ├─ EndRenderPass（IsMergedRenderPassEnd の場合のみ）
       └─ BarrierBatchEnd を Submit（RHI Transition End）
   ```

### 関与クラス・関数一覧

| クラス / 関数 | ファイル | 役割 |
|-------------|--------|------|
| `FRDGBuilder::AddPass()` | `RenderGraphBuilder.h` | パス宣言のエントリ |
| `FRDGBuilder::SetupParameterPass()` | `RenderGraphBuilder.cpp:2624` | パラメータ走査・リソース依存収集 |
| `FRDGBuilder::SetupPassDependencies()` | `RenderGraphBuilder.cpp:2319` | 全パスの First/Last パス確定 |
| `FRDGBuilder::ExecutePasses()` | `RenderGraphBuilder.cpp:2036` | パス実行ループ |
| `FRDGBuilder::ExecutePass()` | `RenderGraphBuilder.cpp:3482` | 単一パスの実行制御 |
| `FRDGBuilder::ExecuteSerialPass()` | `RenderGraphBuilder.cpp:3497` | シリアル実行（Inline モード） |
| `FRDGBuilder::ExecutePassPrologue()` | `RenderGraphBuilder.cpp:3395` | バリア Begin・RenderPass 開始 |
| `FRDGBuilder::ExecutePassEpilogue()` | `RenderGraphBuilder.cpp:3428` | バリア End・RenderPass 終了 |
| `TRDGLambdaPass` | `RenderGraphPass.h` | [[ref_rdg_pass]] ラムダ保持パス型 |
| `FRDGPassRegistry` | `RenderGraphPass.h` | [[ref_rdg_pass]] パス登録コンテナ |
