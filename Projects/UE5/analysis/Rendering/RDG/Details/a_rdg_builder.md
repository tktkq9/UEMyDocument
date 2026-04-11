# A: FRDGBuilder — グラフ構築と実行

- 対象: `RenderGraphBuilder.h/.inl`
- 上位: [[10_rdg_overview]]
- Reference: [[ref_rdg_builder]]

---

## FRDGBuilder とは

レンダーグラフ全体を管理するクラス。スタック上に生成し、`Execute()` で実行する。  
コンストラクタに `FRHICommandListImmediate&` を渡して紐付ける。

```cpp
// 典型的な使用パターン
FRDGBuilder GraphBuilder(
    RHICmdListImmediate,
    RDG_EVENT_NAME("MyRenderGraph"),
    ERDGBuilderFlags::Parallel);

// ... AddPass, CreateTexture 等 ...

GraphBuilder.Execute();
// ← Execute 後は Builder を破棄するだけ。クリーンアップも自動
```

---

## 並列化フラグ（ERDGBuilderFlags）

```cpp
enum class ERDGBuilderFlags {
    None            = 0,
    ParallelSetup   = 1 << 0,  // AddSetupTask の並列化
    ParallelCompile = 1 << 1,  // グラフコンパイルの並列化
    ParallelExecute = 1 << 2,  // パス実行の並列化
    Parallel = ParallelSetup | ParallelCompile | ParallelExecute,
};
```

---

## リソース生成

### テクスチャ

```cpp
// 新規 RDG テクスチャを作成（GPU メモリ確保はグラフコンパイル時）
FRDGTextureRef Tex = GraphBuilder.CreateTexture(
    FRDGTextureDesc::Create2D(
        FIntPoint(1920, 1080),
        PF_FloatRGBA,
        FClearValueBinding::Black,
        TexCreate_RenderTargetable | TexCreate_ShaderResource),
    TEXT("MyRenderTarget"),
    ERDGTextureFlags::None);

// 外部リソース（既存 IPooledRenderTarget）を登録
FRDGTextureRef ExternalTex = GraphBuilder.RegisterExternalTexture(
    SceneRenderTargets.SceneColor,
    ERDGTextureFlags::MultiFrame);  // フレーム間で保持
```

### バッファ

```cpp
// 新規バッファ（要素数固定）
FRDGBufferRef Buf = GraphBuilder.CreateBuffer(
    FRDGBufferDesc::CreateStructuredDesc(sizeof(FMyData), ElementCount),
    TEXT("MyBuffer"));

// 新規バッファ（要素数をコールバックで後決め）
FRDGBufferRef DynBuf = GraphBuilder.CreateBuffer(
    FRDGBufferDesc::CreateStructuredDesc(sizeof(FMyData), 1),
    TEXT("DynBuffer"),
    [&CountBuffer]() -> uint32 {
        // Execute 直前に呼ばれる
        return ReadCountFromGPU(CountBuffer);
    });

// 外部バッファを登録
FRDGBufferRef ExtBuf = GraphBuilder.RegisterExternalBuffer(MyPooledBuffer);
```

### View (SRV / UAV)

```cpp
// テクスチャ SRV
FRDGTextureSRVRef SRV = GraphBuilder.CreateSRV(
    FRDGTextureSRVDesc::Create(Tex));

// バッファ UAV（PixelFormat 指定）
FRDGBufferUAVRef UAV = GraphBuilder.CreateUAV(
    FRDGBufferUAVDesc(Buf, PF_R32_UINT));

// ショートカット版
FRDGTextureUAVRef TexUAV = GraphBuilder.CreateUAV(Tex);
```

### Uniform Buffer

```cpp
// パラメータ構造体を Uniform Buffer として作成
auto* Params = GraphBuilder.AllocParameters<FMyUniformParams>();
Params->SomeValue = 42.0f;
TRDGUniformBufferRef<FMyUniformParams> UB = GraphBuilder.CreateUniformBuffer(Params);
```

---

## パスの追加

### 通常パス（AddPass）

```cpp
// パラメータ構造体を宣言（BEGIN_SHADER_PARAMETER_STRUCT マクロ使用）
auto* PassParams = GraphBuilder.AllocParameters<FMyPassParams>();
PassParams->SceneColor = InputTex;
PassParams->Output = GraphBuilder.CreateUAV(OutputTex);

// ラムダの引数型でパスの種類が決まる
GraphBuilder.AddPass(
    RDG_EVENT_NAME("MyComputePass %dx%d", Width, Height),
    PassParams,
    ERDGPassFlags::Compute,                       // Compute パス
    [PassParams, CS = MyComputeShader](FRHIComputeCommandList& RHICmdList)
    {
        FComputeShaderUtils::Dispatch(RHICmdList, CS, *PassParams,
            FIntVector(GroupX, GroupY, 1));
    });

// Raster パス（FRHICommandList& を使う）
GraphBuilder.AddPass(
    RDG_EVENT_NAME("MyRasterPass"),
    PassParams,
    ERDGPassFlags::Raster,
    [PassParams](FRHICommandList& RHICmdList)
    {
        RHICmdList.DrawIndexedPrimitive(...);
    });
```

### Dispatch パス（並列コマンドリスト）

```cpp
// タスクグラフから複数のコマンドリストを並列録画する高度なAPI
GraphBuilder.AddDispatchPass(
    RDG_EVENT_NAME("ParallelRasterPass"),
    PassParams,
    ERDGPassFlags::Raster,
    [](FRDGDispatchPassBuilder& DispatchBuilder)
    {
        // 並列タスク内で FRHICommandList を取得・録画
        TArray<FRHICommandList*> CmdLists = ...;
        for (auto* CmdList : CmdLists)
        {
            CmdList->DrawIndexedPrimitive(...);
            CmdList->FinishRecording();
        }
    });
```

### パラメータなしパス（デバッグ用）

```cpp
// 既存 RHI コードを一時的にグラフに組み込む際に使用
// NeverCull と SkipRenderPass が自動的に付与される
GraphBuilder.AddPass(
    RDG_EVENT_NAME("LegacyPass"),
    ERDGPassFlags::Raster,
    [](FRHICommandList& RHICmdList)
    {
        // RDG 管理外のコード
    });
```

---

## セットアップタスク（並列前処理）

```cpp
// グラフ実行前に並列で走るセットアップ処理
UE::Tasks::FTask SetupTask = GraphBuilder.AddSetupTask(
    [&MyData]()
    {
        // CPU 側の前処理（DrawList 構築等）
        MyData.Prepare();
    },
    UE::Tasks::ETaskPriority::High);
```

---

## リソース抽出（Execute 後に外部で使う）

```cpp
TRefCountPtr<IPooledRenderTarget> ExtractedTexture;
GraphBuilder.QueueTextureExtraction(RDGTexture, &ExtractedTexture);

// Execute 後に ExtractedTexture が有効になる
GraphBuilder.Execute();
// ← この時点で ExtractedTexture.IsValid() == true
```

---

## グラフコンパイル・実行フロー

```
FRDGBuilder::Execute() 内部:
  1. Compile()
     a. パスの参照カウントを計算（未参照パスをカリング）
     b. リソースバリアを挿入（ERHIAccess の遷移）
     c. 連続 Raster パスを RenderPass にマージ
     d. AsyncCompute の境界を決定
  2. Execute()
     a. パスを順番に実行（ラムダを呼び出す）
     b. Parallel フラグ時は Task グラフで並列実行
  3. PostExecuteCallbacks を呼び出し
  4. FRDGPooledBuffer / IPooledRenderTarget への抽出を完了
```

---

## メモリアロケーション API

```cpp
// グラフのライフタイムに紐付いたメモリ（Execute 後に解放）
void* RawMem   = GraphBuilder.Alloc(1024, 16);
int32* Ints    = GraphBuilder.AllocPODArray<int32>(Count);
auto* Obj      = GraphBuilder.AllocObject<FMyClass>(Arg1, Arg2);
auto& Arr      = GraphBuilder.AllocArray<FMyStruct>();
auto* Params   = GraphBuilder.AllocParameters<FMyPassParams>();
```

---

## 関連リファレンス

| リファレンス | 対象ソース |
|------------|----------|
| [[ref_rdg_builder]] | `RenderGraphBuilder.h/.inl/.cpp` |
| [[ref_rdg_definitions]] | `RenderGraphDefinitions.h`, `RenderGraphFwd.h` |
