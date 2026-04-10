# RDG (Render Dependency Graph) 全体概要

- 取得日: 2026-04-10
- 対象: `D:\UnrealEngine\Engine\Source\Runtime\RenderCore\Public\RenderGraph*.h`
- 上位: [[01_rendering_overview]]
- Details: [[a_rdg_builder]] | [[b_rdg_resources]] | [[c_rdg_pass]] | [[d_rdg_parameters]]
- Reference: [[ref_rdg_builder]] | [[ref_rdg_resources]] | [[ref_rdg_pass]] | [[ref_rdg_utils]]

---

## RDG とは

**Render Dependency Graph**。UE5 のレンダリングパス全体を DAG（有向非巡回グラフ）として宣言的に記述し、  
バリア・メモリライフタイム・並列実行・リソースカリングを自動管理するフレームワーク。

`FRDGBuilder` に「どのリソースを使ってどの処理をするか」を宣言（AddPass）するだけで、  
実際の RHI 命令列はグラフコンパイル後に最適化されて発行される。

---

## 全体アーキテクチャ

```mermaid
graph TD
    App["レンダーコード\n(AddPass の呼び出し)"] --> Builder

    subgraph Builder["FRDGBuilder"]
        CreateRes["CreateTexture / CreateBuffer\nリソース宣言"]
        AddPass["AddPass\nパス宣言 (ラムダ + パラメータ構造体)"]
        Compile["Execute 前\nグラフコンパイル"]
        Execute["Execute\nRHI コマンドを発行"]
        CreateRes --> AddPass --> Compile --> Execute
    end

    Builder --> RHI["RHI Layer\n(DX12 / Vulkan / Metal)"]

    subgraph Compile
        Cull["未使用パスのカリング"]
        Barrier["リソースバリア挿入"]
        Merge["RenderPass マージ"]
        Async["AsyncCompute 領域分割"]
    end
```

---

## フレームの流れ（概略）

```
FRDGBuilder GraphBuilder(RHICmdListImmediate, RDG_EVENT_NAME("Frame"));

// 1. 外部リソースを登録
FRDGTextureRef SceneColor = GraphBuilder.RegisterExternalTexture(SceneColorRT);

// 2. 新規リソースを作成（宣言のみ・GPUメモリはまだ確保されない）
FRDGTextureRef OutputTexture = GraphBuilder.CreateTexture(
    FRDGTextureDesc::Create2D(Resolution, PF_FloatRGBA, FClearValueBinding::Black,
        TexCreate_RenderTargetable | TexCreate_ShaderResource),
    TEXT("MyOutput"));

// 3. パスを追加（ラムダは defer される）
auto* PassParams = GraphBuilder.AllocParameters<FMyPassParameters>();
PassParams->InTexture = SceneColor;
PassParams->OutTexture = GraphBuilder.CreateUAV(OutputTexture);

GraphBuilder.AddPass(
    RDG_EVENT_NAME("MyComputePass"),
    PassParams,
    ERDGPassFlags::Compute,
    [PassParams, MyCS](FRHIComputeCommandList& RHICmdList) {
        FComputeShaderUtils::Dispatch(RHICmdList, MyCS, *PassParams, GroupCount);
    });

// 4. Execute（コンパイル → バリア挿入 → 実行）
GraphBuilder.Execute();
```

---

## 主要コンセプト

| コンセプト | 説明 |
|-----------|------|
| **宣言的記述** | AddPass でパスを宣言するだけ。実行順序・バリアは自動決定 |
| **リソースカリング** | どのパスからも参照されないリソース/パスは自動的に除外 |
| **RenderPass マージ** | 連続する Raster パスを自動マージしてバリアを削減 |
| **AsyncCompute** | `ERDGPassFlags::AsyncCompute` でグラフィックスと並走 |
| **並列実行** | `ERDGBuilderFlags::Parallel` で Setup/Compile/Execute を並列化 |
| **Blackboard** | `FRDGBlackboard` でパス間データを型安全に受け渡し |
| **Transient Resource** | フレーム内のみ存在するリソースはエイリアシングで再利用 |

---

## 主要クラス一覧

| クラス | 役割 |
|--------|------|
| `FRDGBuilder` | グラフ全体の管理・実行 |
| `FRDGTexture` | RDG 管理テクスチャ |
| `FRDGBuffer` | RDG 管理バッファ |
| `FRDGPass` | パスの基底クラス |
| `FRDGBlackboard` | パス間データストア |
| `FRDGEventName` | GPU プロファイル名 |
| `FRDGParameterStruct` | パスパラメータのラッパー |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RDG.Debug` | 0 | RDG デバッグ情報出力 |
| `r.RDG.Parallel` | 1 | 並列 Setup/Compile/Execute |
| `r.RDG.ImmediateMode` | 0 | 即時実行モード（デバッグ用）|
| `r.RDG.BreakPoint` | 0 | 特定パスでブレーク |
| `r.RDG.ClobberResources` | 0 | 未初期化リソースをランダム値で埋める |
| `r.RDG.TransientResourceAllocator` | 1 | Transient リソース再利用 |

---

## 主要ソースファイル一覧

| ファイル | 役割 |
|---------|------|
| `RenderGraphBuilder.h/.inl` | `FRDGBuilder` — グラフ構築・実行 |
| `RenderGraphResources.h/.inl` | `FRDGTexture`, `FRDGBuffer`, View 型 |
| `RenderGraphPass.h` | `FRDGPass`, `TRDGLambdaPass` |
| `RenderGraphDefinitions.h` | 基本 enum・型エイリアス |
| `RenderGraphBlackboard.h` | `FRDGBlackboard` |
| `RenderGraphEvent.h/.inl` | `FRDGEventName`, スコープマクロ |
| `RenderGraphParameter.h` | `FRDGParameterStruct` |
| `RenderGraphUtils.h` | ユーティリティ関数群 |
| `RenderGraphAllocator.h` | グラフアロケータ |
| `RenderGraphValidation.h` | デバッグ検証ロジック |
| `RenderGraphTrace.h` | RDG トレース出力 |
| `RenderGraph.h` | 上記を一括 include |
