# リファレンス：RenderGraphBuilder.h / RenderGraphBuilder.inl / RenderGraphBuilder.cpp

- グループ: a - Builder
- 上位: [[a_rdg_builder]]
- 関連: [[ref_rdg_definitions]] | [[ref_rdg_pass]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphBuilder.h`
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphBuilder.inl`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphBuilder.cpp`

## 概要

レンダーグラフ全体を管理するメインクラス `FRDGBuilder` の全 API 定義。  
`FRDGScopeState`（イベントスコープ管理基底）を継承する。  
グラフの構築（`AddPass`）・コンパイル（`Compile`）・実行（`Execute`）を担う。

---

## FRDGBuilder

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Blackboard` | `FRDGBlackboard` | パス間データストア（**公開**） |
| `BuilderName` | `FRDGEventName` | GPU プロファイル上のグラフ名 |
| `ProloguePass` | `FRDGPass*` | グラフ先頭のセンチネルパス（プリバリア用） |
| `EpiloguePass` | `FRDGPass*` | グラフ末尾のセンチネルパス（抽出バリア用） |
| `Passes` | `FRDGPassRegistry` | 全パスの格納レジストリ |
| `Textures` | `FRDGTextureRegistry` | 全テクスチャの格納レジストリ |
| `Buffers` | `FRDGBufferRegistry` | 全バッファの格納レジストリ |
| `Views` | `FRDGViewRegistry` | 全 SRV/UAV の格納レジストリ |
| `UniformBuffers` | `FRDGUniformBufferRegistry` | 全 Uniform Buffer の格納レジストリ |
| `ExternalTextures` | `TRobinHoodHashMap<FRHITexture*, FRDGTexture*>` | 外部テクスチャの重複登録防止マップ |
| `ExternalBuffers` | `TRobinHoodHashMap<FRHIBuffer*, FRDGBuffer*>` | 外部バッファの重複登録防止マップ |
| `ExtractedTextures` | `TArray<FExtractedTexture>` | `QueueTextureExtraction` で登録された抽出リスト |
| `ExtractedBuffers` | `TArray<FExtractedBuffer>` | `QueueBufferExtraction` で登録された抽出リスト |
| `TransientResourceAllocator` | `IRHITransientResourceAllocator*` | トランジェントメモリアロケータ |
| `bSupportsAsyncCompute` | `bool` | AsyncCompute 対応プラットフォームか |
| `bSupportsTransientTextures` | `bool` | トランジェントテクスチャを使えるか |
| `bSupportsTransientBuffers` | `bool` | トランジェントバッファを使えるか |
| `AsyncComputePassCount` | `uint32` | AsyncCompute パスの数 |
| `RasterPassCount` | `uint32` | Raster パスの数 |

---

## コンストラクタ

```cpp
FRDGBuilder(
    FRHICommandListImmediate& RHICmdList,
    FRDGEventName Name = {},
    ERDGBuilderFlags Flags = ERDGBuilderFlags::None,
    EShaderPlatform ShaderPlatform = GMaxRHIShaderPlatform);
```

### パラメータ

| 引数 | 型 | 説明 |
|-----|-----|------|
| `RHICmdList` | `FRHICommandListImmediate&` | レンダースレッドの即時コマンドリスト |
| `Name` | `FRDGEventName` | GPU プロファイラに表示されるグラフ名（`RDG_EVENT_NAME(...)` で生成） |
| `Flags` | `ERDGBuilderFlags` | 並列実行の有効化フラグ（`::Parallel` を推奨） |
| `ShaderPlatform` | `EShaderPlatform` | シェーダープラットフォーム判定に使用 |

### 使用箇所
- `FDeferredShadingSceneRenderer::Render()` — フレームのメイングラフを生成
- `FPostProcessing::AddPasses()` — ポストプロセスグラフを生成

---

## Execute()

グラフのコンパイルと実行を行うメインエントリポイント（`RenderGraphBuilder.cpp:1755`）。

```cpp
void FRDGBuilder::Execute();
```

### 内部処理フロー

```
Execute() (RenderGraphBuilder.cpp:1755)
  │
  ├─ FlushAccessModeQueue()              ← UseExternalAccessMode の遅延処理
  ├─ SetupEmptyPass(EpiloguePass)         ← 末尾センチネルパス生成
  ├─ WaitForParallelSetupTasks(Compile)   ← AddSetupTask の完了待ち
  │
  ├─ [非即時モード]
  │   ├─ Compile()                        ← グラフコンパイル（後述）
  │   ├─ AllocatePooledTextures()         ← プールからテクスチャを確保
  │   ├─ AllocatePooledBuffers()          ← プールからバッファを確保
  │   ├─ AllocateTransientResources()     ← トランジェントメモリを確保
  │   └─ CollectPassBarriers()            ← バリア情報の収集
  │
  ├─ ExecutePasses()                      ← 全パスのラムダを順番に実行
  │   ├─ ExecutePassPrologue()            ← バリア発行・RenderPass 開始
  │   ├─ [Lambda 実行]
  │   └─ ExecutePassEpilogue()            ← RenderPass 終了・後バリア
  │
  ├─ 抽出処理                             ← ExtractedTextures/Buffers を解決
  └─ PostExecuteCallbacks                 ← AddPostExecuteCallback のコールバック群
```

### 使用箇所
- あらゆる `FRDGBuilder` の末尾で必ず 1 回呼ばれる

---

## Compile()（private）

パス依存グラフを解析して最適化する内部コンパイルフェーズ（`RenderGraphBuilder.cpp:1316`）。

### 内部処理フロー

```
Compile() (RenderGraphBuilder.cpp:1316)
  │
  ├─ 1. SetupPassDependencies(Pass)       ← 全パスのリソース参照を走査
  │      各パスが読み書きするリソースの参照カウントを加算
  │
  ├─ 2. カリング (GRDGCullPasses > 0)
  │      EpiloguePass から後ろ向き DFS でアクセス可能なパスを発見
  │      到達できないパスを bCulled = true にマーク
  │
  ├─ 3. RenderPass マージ
  │      連続する Raster パスが同じ RT を使う場合に 1 RenderPass に統合
  │      bIsMergedRenderPassBegin / End フラグを設定
  │
  └─ 4. AsyncCompute 領域決定
         AsyncCompute パスのフェンス挿入位置（Graphics/AsyncCompute 境界）を計算
```

---

## AddPass()

```cpp
template <typename ParameterStructType, typename ExecuteLambdaType>
FRDGPassRef AddPass(
    FRDGEventName&& Name,
    const ParameterStructType* ParameterStruct,
    ERDGPassFlags Flags,
    ExecuteLambdaType&& ExecuteLambda);
```

### パラメータ

| 引数 | 型 | 説明 |
|-----|-----|------|
| `Name` | `FRDGEventName&&` | GPU プロファイル上のパス名 |
| `ParameterStruct` | `const T*` | `AllocParameters<T>()` で確保したパラメータ構造体 |
| `Flags` | `ERDGPassFlags` | パスの種類（`Compute` / `Raster` / `AsyncCompute` 等） |
| `ExecuteLambda` | ラムダ | ラムダ引数型でパスモードが決まる（後述） |

### ラムダ引数型とパスモード

| ラムダ引数型 | 実行モード |
|------------|-----------|
| `FRHICommandListImmediate&` | レンダースレッドでインライン実行（並列化されない） |
| `FRHICommandList&` | `ParallelExecute` 時にタスクグラフで並列実行し、`Execute()` 末尾で待機 |
| `FRDGAsyncTask, FRHICommandList&` | 並列実行し、手動で `WaitForAsyncExecuteTask()` まで待機しない |
| `FRHIComputeCommandList&` | Compute / AsyncCompute パスで使用 |

### 使用箇所
- レンダリングコード内の全 GPU 処理（Compute / Raster / AsyncCompute すべて）

---

## RegisterExternalTexture()

```cpp
FRDGTextureRef RegisterExternalTexture(
    const TRefCountPtr<IPooledRenderTarget>& ExternalPooledTexture,
    ERDGTextureFlags Flags = ERDGTextureFlags::None);
```

### パラメータ

| 引数 | 型 | 説明 |
|-----|-----|------|
| `ExternalPooledTexture` | `TRefCountPtr<IPooledRenderTarget>&` | 前フレームから引き継がれた外部テクスチャ |
| `Flags` | `ERDGTextureFlags` | `MultiFrame` を付けるとフレームをまたいで保持される |

### 内部処理
`ExternalTextures` ハッシュマップで重複チェックされる。  
同じ `FRHITexture*` を 2 回登録しても同じ `FRDGTextureRef` が返る。

### 使用箇所
- `FSceneRenderTargets::RegisterExternalSceneColor()` — SceneColor の登録
- [[ref_rdg_resources]] の `FRDGViewableResource::IsExternal()` が `true` になる

---

## QueueTextureExtraction()

```cpp
void QueueTextureExtraction(
    FRDGTextureRef Texture,
    TRefCountPtr<IPooledRenderTarget>* OutPooledTexturePtr,
    ERDGResourceExtractionFlags Flags = ERDGResourceExtractionFlags::None);
```

### パラメータ

| 引数 | 型 | 説明 |
|-----|-----|------|
| `Texture` | `FRDGTextureRef` | 抽出するグラフ内テクスチャ |
| `OutPooledTexturePtr` | `TRefCountPtr<IPooledRenderTarget>*` | Execute() 後に有効になる出力先ポインタ |
| `Flags` | `ERDGResourceExtractionFlags` | 抽出オプション（現在は `None` のみ） |

### 使用箇所
- `FSceneRenderTargets` が次フレームに引き継ぐテクスチャの抽出
- `FPostProcessing` 系での中間バッファの保持

---

## AddSetupTask()

並列セットアップタスクを登録する関数群（`RenderGraphBuilder.h:256`）。

```cpp
template <typename TaskLambda>
UE::Tasks::FTask AddSetupTask(
    TaskLambda&& Task,
    bool bCondition = true,
    ERDGSetupTaskWaitPoint WaitPoint = ERDGSetupTaskWaitPoint::Compile);
```

### パラメータ（全オーバーロード共通）

| 引数 | 型 | 説明 |
|-----|-----|------|
| `Task` | ラムダ | 並列実行する CPU 前処理（DrawList 構築等） |
| `bCondition` | `bool` | false の場合は即時実行（非並列） |
| `WaitPoint` | `ERDGSetupTaskWaitPoint` | `Compile` = コンパイル前に待機、`Execute` = 実行前に待機 |
| `Priority` | `UE::Tasks::ETaskPriority` | タスク優先度（デフォルト `Normal`） |
| `Pipe` | `UE::Tasks::FPipe*` | タスクの直列化パイプ |
| `Prerequisites` | コレクション | 先行タスクの `FTask` 一覧 |

### 使用箇所
- `FDeferredShadingSceneRenderer::GatherDynamicMeshElements()` — メッシュ DrawList の並列構築

---

## UseExternalAccessMode() / UseInternalAccessMode()

```cpp
void UseExternalAccessMode(
    FRDGViewableResource* Resource,
    ERHIAccess ReadOnlyAccess,
    ERHIPipeline Pipelines = ERHIPipeline::Graphics);

void UseInternalAccessMode(FRDGViewableResource* Resource);
```

RDG のバリア管理を外部から制御するための API。  
`UseExternalAccessMode` を呼ぶと、それ以降のパスでそのリソースの RHI にアクセスできる。  
必ず `UseInternalAccessMode` で元に戻すこと。

### 使用箇所
- VSM / Shadow Map のテクスチャを RDG 外の RHI コードで直接読む場合

---

## QueueBufferUpload()

```cpp
void QueueBufferUpload(
    FRDGBufferRef Buffer,
    const void* InitialData,
    uint64 InitialDataSize,
    ERDGInitialDataFlags InitialDataFlags = ERDGInitialDataFlags::None);
```

### パラメータ

| 引数 | 型 | 説明 |
|-----|-----|------|
| `Buffer` | `FRDGBufferRef` | アップロード先のバッファ |
| `InitialData` | `const void*` | コピー元 CPU データポインタ |
| `InitialDataSize` | `uint64` | コピーバイト数 |
| `InitialDataFlags` | `ERDGInitialDataFlags` | `NoCopy` = データポインタを保持（コピーしない） |

### 使用箇所
- 頂点バッファや定数バッファの初期データアップロード

---

## マイナー関数

> [!note]- ConvertToExternalTexture / ConvertToExternalBuffer — グラフ内リソースを即時外部に昇格
>
> ```cpp
> const TRefCountPtr<IPooledRenderTarget>& ConvertToExternalTexture(FRDGTextureRef Texture);
> const TRefCountPtr<FRDGPooledBuffer>& ConvertToExternalBuffer(FRDGBufferRef Buffer);
> ```
>
> グラフ内で作成したリソースを即時 GPU メモリに確保し、外部リソースとして扱えるようにする。  
> メモリプレッシャーが増加するため、段階的な RDG 移行作業時のみ使用すること。

> [!note]- SkipInitialAsyncComputeFence — AsyncCompute フェンス省略
>
> ```cpp
> void SkipInitialAsyncComputeFence();
> ```
>
> デフォルトでは前の RDG ビルダーとの AsyncCompute オーバーラップを防ぐフェンスが発行される。  
> この関数を呼ぶとそのフェンスを省略できる（前ビルダーとの依存関係がない場合に最適化として使用）。

> [!note]- SetFlushResourcesRHI — RHI リソースの解放
>
> ```cpp
> void SetFlushResourcesRHI();
> ```
>
> 未使用の RHI リソースを解放するよう指示する。  
> 即時モードでは即時実行、通常モードでは Execute() 前に処理される。

> [!note]- TickPoolElements — フレームごとのプールメンテナンス
>
> ```cpp
> static void TickPoolElements();
> ```
>
> フレームの終わりにバッファプール・テクスチャプールの未使用エントリを解放する静的関数。  
> `FRDGBuilder::Execute()` の内部で自動的に呼ばれる（手動呼び出し不要）。

> [!note]- GetPooledTexture / GetPooledBuffer — 外部リソースへの直接アクセス
>
> ```cpp
> const TRefCountPtr<IPooledRenderTarget>& GetPooledTexture(FRDGTextureRef Texture) const;
> const TRefCountPtr<FRDGPooledBuffer>& GetPooledBuffer(FRDGBufferRef Buffer) const;
> ```
>
> 外部登録またはすでに抽出済みのリソースにのみ有効。  
> グラフ内のみで使うリソースには呼べない。

---

## ERDGBuilderFlags

```cpp
enum class ERDGBuilderFlags {
    None            = 0,
    ParallelSetup   = 1 << 0,  // AddSetupTask を並列タスクで実行
    ParallelCompile = 1 << 1,  // Compile() の一部を並列化
    ParallelExecute = 1 << 2,  // FRHICommandList& パスを並列実行
    Parallel = ParallelSetup | ParallelCompile | ParallelExecute,
};
```

### 使用箇所
- `FDeferredShadingSceneRenderer::Render()` — `ERDGBuilderFlags::Parallel` で本番グラフを生成
