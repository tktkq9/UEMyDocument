# RHI 全体概要

- 取得日: 2026-04-12
- 対象:
  - `D:\UnrealEngine\Engine\Source\Runtime\RHI\`
  - `D:\UnrealEngine\Engine\Source\Runtime\D3D12RHI\`
  - `D:\UnrealEngine\Engine\Source\Runtime\RenderCore\` (RDG)
- 上位: [[01_rendering_overview]]
- Details: [[a_rhi_abstraction]] | [[b_rhi_resources]] | [[c_rhi_d3d12_device]] | [[d_rhi_d3d12_context]] | [[e_rhi_rdg_bridge]]
- Reference: [[ref_rhi_commandlist]] | [[ref_rhi_resources]] | [[ref_rhi_dynamic_rhi]] | [[ref_rhi_d3d12_device]] | [[ref_rhi_d3d12_context]] | [[ref_rhi_d3d12_resources]]

---

## RHI とは

**Rendering Hardware Interface**（RHI）は UE5 のレンダリングコードと GPU API を切り離す抽象層。  
レンダラーは RHI の API のみを呼び出し、実行時にロードされるプラグイン（D3D12RHI・VulkanRHI・MetalRHI 等）が
プラットフォーム固有の処理に変換する。

---

## 全体アーキテクチャ

```
┌─────────────────────────────────────────────────────────────┐
│               レンダラー / RDG (FRDGBuilder)                 │  ← パス記述（高レベル）
│     AddPass() でラムダを登録、依存グラフを構築               │
└────────────────────────┬────────────────────────────────────┘
                         │ Execute() 時に変換
┌────────────────────────▼────────────────────────────────────┐
│            FRHICommandList / FRHICommandListImmediate        │  ← RHI 抽象コマンドキュー
│  SetGraphicsPipelineState / DrawIndexedPrimitive 等を記録    │
└────────────────────────┬────────────────────────────────────┘
                         │ Flush / Submit
┌────────────────────────▼────────────────────────────────────┐
│              IRHICommandContext（純粋仮想）                   │  ← プラットフォーム抽象
│  RHI_API SetGraphicsPipelineState() 等の仮想関数             │
└────────────────────────┬────────────────────────────────────┘
                         │ 実装
┌────────────────────────▼────────────────────────────────────┐
│         FD3D12ContextCommon（D3D12RHI プラグイン）            │  ← D3D12 実装
│  ID3D12GraphicsCommandList に変換して記録                     │
└────────────────────────┬────────────────────────────────────┘
                         │ Submit
┌────────────────────────▼────────────────────────────────────┐
│            FD3D12Queue / FD3D12Submission                    │  ← GPU キュー投入
│  ID3D12CommandQueue::ExecuteCommandLists()                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 主要モジュール構成

| モジュール | ディレクトリ | 役割 |
|-----------|------------|------|
| `RHI` | `Runtime/RHI/` | 抽象 API・リソース定義・コマンドリスト |
| `RenderCore` | `Runtime/RenderCore/` | RDG・シェーダーコア・スレッド制御 |
| `D3D12RHI` | `Runtime/D3D12RHI/` | DirectX 12 実装プラグイン |
| `VulkanRHI` | `Runtime/VulkanRHI/` | Vulkan 実装（Linux / Android / Mac 対応） |
| `MetalRHI` | `Runtime/Apple/MetalRHI/` | Metal 実装（iOS / macOS） |

---

## コード実行フロー

```
[起動時 — D3D12RHI モジュールロード]
FD3D12DynamicRHI::FD3D12DynamicRHI()     D3D12RHI.cpp:60
  └─ FD3D12DynamicRHI::PostInit()         D3D12RHI.cpp:257
       └─ FD3D12Device::Initialize()       D3D12Device.cpp
            ├─ CreateCommandQueue() × 3   (Graphics / AsyncCompute / Copy)
            └─ FD3D12DescriptorManager::Init()

[フレームごと — コマンド記録]
FRHICommandListImmediate (GRHICommandList)
  └─ AddPass() / SetGraphicsPipelineState() / DrawIndexedPrimitive()
       ↓ EnqueueCommand (コマンドをメモリスタックに積む)
  └─ ImmediateFlush()                       RHICommandList.cpp:1573
       └─ FRHICommandListBase::Execute()     RHICommandList.cpp:518
            └─ IRHICommandContext::RHIXxx() (仮想関数ディスパッチ)
                 └─ FD3D12CommandContext::RHIDrawIndexedPrimitive()  D3D12Commands.cpp:1270
                      ├─ FlushResourceBarriers()  D3D12CommandList.cpp:70
                      └─ CommandList->DrawIndexedInstanced()

[フレーム終了 — GPU 投入]
FD3D12DynamicRHI::RHIEndFrame()           D3D12RHI.cpp:572
  └─ FD3D12Submission (Submission Thread)  D3D12Submission.cpp:553
       └─ ID3D12CommandQueue::ExecuteCommandLists()  D3D12Submission.cpp:827
            └─ ID3D12CommandQueue::Signal() (フェンスをシグナル)

[RDG 経由の場合]
FRDGBuilder::Execute()                    RenderGraphBuilder.cpp:1755
  ├─ Compile() (依存解析・バリア計算)      RenderGraphBuilder.cpp:1316
  ├─ AllocateResources (FRDGTexture→FRHITexture 確定)
  └─ ExecutePass() × n                    RenderGraphBuilder.cpp:3482
       ├─ ExecutePassPrologue() (バリア発行) RenderGraphBuilder.cpp:3395
       └─ ラムダ呼び出し → FRHICommandList::DrawXxx / DispatchXxx
```

---

## RHI の 3 階層

### 1. 抽象層（`Runtime/RHI/`）

```
FRHIResource                      ← 全 RHI リソースの基底（参照カウント付き）
  ├── FRHITexture                 ← テクスチャ
  ├── FRHIBuffer                  ← バッファ（VB / IB / UB / Structured 等）
  ├── FRHIShader (各種)           ← VS / PS / CS 等
  ├── FRHIGraphicsPipelineState   ← Graphics PSO
  └── FRHIComputePipelineState    ← Compute PSO

FRHICommandListBase               ← コマンドリスト基底
  └── FRHICommandList             ← 遅延実行コマンドリスト（マルチスレッド記録用）
        └── FRHICommandListImmediate ← 即時実行型（メインレンダースレッド用）

IRHICommandContext                ← プラットフォーム実装が持つ仮想関数インターフェース
FDynamicRHI                       ← リソース生成・破棄の仮想インターフェース
```

### 2. D3D12 実装層（`Runtime/D3D12RHI/`）

```
FD3D12DynamicRHI                  ← FDynamicRHI の D3D12 実装（モジュール起点）
  └── FD3D12Adapter               ← IDXGIAdapter 単位の管理
        └── FD3D12Device          ← ID3D12Device ラッパー（ディスクリプタヒープ等管理）
              ├── FD3D12Queue     ← ID3D12CommandQueue（Graphics / Compute / Copy）
              └── FD3D12ContextCommon ← IRHICommandContext の D3D12 実装
                    └── FD3D12CommandList ← ID3D12GraphicsCommandList ラッパー
```

### 3. RDG（`Runtime/RenderCore/`）

```
FRDGBuilder                       ← パス登録・依存解析・バリア自動生成
  ├── AddPass(…)                  → FRDGPass を生成してグラフに追加
  └── Execute()                   → FRHICommandListImmediate に対してパスを順番に実行
        └── FRDGPass::Execute()   → ラムダ内で FRHICommandList を直接呼び出す
```

---

## RDG と RHI の関係

```
                       コード記述レベル
                       ┌─────────────────────────────┐
                       │  AddPass([](FRHICommandList& │
                       │  RHICmdList) {               │  ← RDG ラムダ内で RHI を呼ぶ
                       │    RHICmdList.DrawXxx();     │
                       │  });                         │
                       └──────────────┬──────────────┘
                                      │ FRDGBuilder::Execute()
                       ┌──────────────▼──────────────┐
                       │  FRHICommandListImmediate    │  ← RDG が内部で使う RHI コマンドリスト
                       │  （= GRHICommandList）       │
                       └──────────────┬──────────────┘
                                      │ RHI スレッドに Enqueue
                       ┌──────────────▼──────────────┐
                       │  IRHICommandContext           │  ← D3D12 実装が受け取る
                       │  （FD3D12ContextCommon）      │
                       └─────────────────────────────┘
```

**重要な分担**:
- **RDG**: リソースライフタイム・バリア・パス依存を管理。`FRDGTexture` / `FRDGBuffer` は仮想リソース
- **RHI**: 実際の GPU リソース（`FRHITexture*` / `FRHIBuffer*`）と GPU コマンドを管理
- RDG の `Execute()` 中に仮想リソースが実 RHI リソースに確定し、バリアが挿入される

---

## スレッドモデル

```
Game Thread          Render Thread          RHI Thread
     │                    │                    │
     │──Proxy更新──→      │                    │
     │                    │──AddPass──→ RDG   │
     │                    │──Execute──→ RHI CmdList に変換
     │                    │                    │──D3D12 コマンド記録
     │                    │                    │──ExecuteCommandLists→ GPU
```

- `r.RHIThread.Enable = 1`（デフォルト）で RHI スレッドが有効
- RHI Thread が `FRHICommandList` のコマンドを `IRHICommandContext` に投入
- RHI Thread が無効の場合は Render Thread が直接 D3D12 コマンドを記録

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RHIThread.Enable` | 1 | RHI 専用スレッド有効（0=レンダースレッドが直接実行）|
| `r.D3D12.MaxCommandsPerCommandList` | 10000 | コマンドリスト分割閾値 |
| `r.D3D12.UseD3DDebugDevice` | 0 | D3D12 デバッグレイヤー有効 |
| `r.D3D12.EnableDRED` | 0 | GPU クラッシュ診断（DRED）有効 |
| `r.D3D12.GPUTimeout` | 1 | GPU タイムアウト検出 |
| `r.Vulkan.EnableValidation` | 0 | Vulkan バリデーションレイヤー |
| `r.RHI.RayTracing` | 1 | レイトレーシング有効 |
