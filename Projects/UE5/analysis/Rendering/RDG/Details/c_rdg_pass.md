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
