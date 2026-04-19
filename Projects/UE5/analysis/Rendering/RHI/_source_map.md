# RHI ソースマップ

- 対象: Rendering Hardware Interface（抽象層 + D3D12 実装）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[12_rhi_overview]]

RHI はレンダラーと GPU API の境界。抽象層（`Runtime/RHI/`）と
プラットフォーム実装（`D3D12RHI` / `VulkanRHI` / `MetalRHI`）に分かれる。

---

## ソースパス

| 対象 | パス |
|------|------|
| RHI 抽象層 | `Engine/Source/Runtime/RHI/` |
| RHI 公開ヘッダ | `Engine/Source/Runtime/RHI/Public/` |
| D3D12 実装 | `Engine/Source/Runtime/D3D12RHI/` |
| Vulkan 実装 | `Engine/Source/Runtime/VulkanRHI/` |
| Metal 実装 | `Engine/Source/Runtime/Apple/MetalRHI/` |

---

## 抽象層（`Runtime/RHI/Public/`）

### コマンドリスト

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `RHICommandList.h` | `FRHICommandListBase`, `FRHICommandList`, `FRHICommandListImmediate` | コマンドリスト基底・即時実行型 | [[Reference/ref_rhi_commandlist]] |
| `RHICommandList.inl` | インライン実装 | Enqueue・ラムダキャプチャ | 同上 |
| `Private/RHICommandList.cpp` | `Execute()`:518, `ImmediateFlush()`:1573 | 実行・フラッシュ | [[Details/a_rhi_abstraction]] |
| `RHIContext.h` | `IRHICommandContext`, `IRHIComputeContext` | プラットフォーム実装の仮想 IF | 同 |

### リソース型

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `RHIResources.h` | `FRHIResource`, `FRHITexture`, `FRHIBuffer`, `FRHIShader`, `FRHIGraphicsPipelineState` | RHI リソース基底・派生型 | [[Reference/ref_rhi_resources]] |
| `RHI.h` | 共通 enum / 型 (`EPixelFormat`, `ETextureCreateFlags` 等) | — | — |
| `RHIGlobals.h` | `GRHISupports*`, feature flags | ランタイム capability | — |
| `RHIDefinitions.h` | `ERHIShaderPlatform`, `EShaderFrequency` | プラットフォーム enum | — |

### Dynamic RHI（プラグインエントリ）

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `DynamicRHI.h` | `FDynamicRHI` | RHI 実装の基底仮想クラス | [[Reference/ref_rhi_dynamic_rhi]] |
| `RHI.cpp` | `RHIInit()`, `GDynamicRHI` | 起動時 RHI 初期化 | — |
| `RHIShaderFormatDefinitions.inl` | シェーダーフォーマット定義 | — | — |

### サポート機能

| ファイル | 主要クラス | 役割 |
|---------|----------|------|
| `RHIUtilities.h` | ヘルパー関数群 | コピー・クリア等ユーティリティ |
| `RHIBreadcrumbs.h` | `FRHIBreadcrumb` | GPU クラッシュ診断用パンくず |
| `RHIPipelineStateCache.h` | `FRHIPipelineStateCache` | PSO キャッシュ |
| `RHITransientResourceAllocator.h` | `IRHITransientResourceAllocator` | RDG 用 Transient アロケータ |

---

## D3D12 実装層（`Runtime/D3D12RHI/`）

### エントリ・デバイス

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `Private/D3D12RHI.cpp` | `FD3D12DynamicRHI` ctor:60, `PostInit()`:257, `RHIEndFrame()`:572 | FDynamicRHI の D3D12 実装・モジュール起点 | [[Details/c_rhi_d3d12_device]] |
| `Private/D3D12Adapter.cpp` | `FD3D12Adapter` | IDXGIAdapter 単位の管理 | 同 |
| `Private/D3D12Device.cpp` | `FD3D12Device::Initialize()` | ID3D12Device ラッパー・キュー作成 | [[Reference/ref_rhi_d3d12_device]] |
| `Private/D3D12Queue.cpp` | `FD3D12Queue` | ID3D12CommandQueue（Graphics/Compute/Copy） | — |

### コマンド記録・実行

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `Private/D3D12CommandContext.cpp` | `FD3D12ContextCommon`, `FD3D12CommandContext` | IRHICommandContext の D3D12 実装 | [[Reference/ref_rhi_d3d12_context]], [[Details/d_rhi_d3d12_context]] |
| `Private/D3D12Commands.cpp` | `RHIDrawIndexedPrimitive()`:1270 等 | Draw / Dispatch 実装 | 同 |
| `Private/D3D12CommandList.cpp` | `FD3D12CommandList`, `FlushResourceBarriers()`:70 | ID3D12GraphicsCommandList ラッパー | — |
| `Private/D3D12Submission.cpp` | `FD3D12Submission::Submit()`:553, `ExecuteCommandLists()`:827 | Submission Thread・GPU 投入 | — |

### リソース

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `Private/D3D12Texture.cpp` | `FD3D12Texture` | D3D12 テクスチャ | [[Reference/ref_rhi_d3d12_resources]] |
| `Private/D3D12Buffer.cpp` | `FD3D12Buffer` | D3D12 バッファ | 同 |
| `Private/D3D12View.cpp` | `FD3D12ShaderResourceView`, `FD3D12UnorderedAccessView` | SRV/UAV/RTV | 同 |
| `Private/D3D12DescriptorCache.cpp` | `FD3D12DescriptorManager`, `FD3D12DescriptorCache` | ディスクリプタヒープ管理 | — |
| `Private/D3D12PipelineState.cpp` | `FD3D12GraphicsPipelineState`, `FD3D12ComputePipelineState` | PSO 管理・キャッシュ | — |

---

## 呼び出しチェーン（Draw 1 回の例）

```
RDG ラムダ内: RHICmdList.DrawIndexedPrimitive(...)
  │
  ├─ FRHICommandList に Enqueue           RHICommandList.h
  │
  └─ ImmediateFlush()                     RHICommandList.cpp:1573
       │
       └─ FRHICommandListBase::Execute()  RHICommandList.cpp:518
            │ (仮想関数ディスパッチ)
            │
            └─ FD3D12CommandContext::RHIDrawIndexedPrimitive()  D3D12Commands.cpp:1270
                 │
                 ├─ FlushResourceBarriers()                      D3D12CommandList.cpp:70
                 └─ ID3D12GraphicsCommandList::DrawIndexedInstanced()

[フレーム終了]
FD3D12DynamicRHI::RHIEndFrame()           D3D12RHI.cpp:572
  └─ FD3D12Submission::Submit()           D3D12Submission.cpp:553
       └─ ID3D12CommandQueue::ExecuteCommandLists()  D3D12Submission.cpp:827
```

---

## 3 スレッドの役割

| スレッド | 扱うファイル | 役割 |
|---------|------------|------|
| Render Thread | `Runtime/Renderer/` + RDG | AddPass・Compile |
| RHI Thread | `RHICommandList.cpp` + `D3D12CommandContext.cpp` | コマンドリスト実行（仮想 → D3D12 API） |
| Submission Thread | `D3D12Submission.cpp` | `ExecuteCommandLists` で GPU 投入 |

有効スイッチ: `r.RHIThread.Enable = 1`（デフォルト）

---

## RHI ⇔ RDG ブリッジ

| ファイル | 役割 | 参照 |
|---------|------|------|
| `RenderGraphBuilder.cpp:AllocateResources()` | RDG 仮想リソース → `FRHITexture*/Buffer*` 確定 | [[Details/e_rhi_rdg_bridge]] |
| `RHITransientResourceAllocator.h` | Transient Allocator IF（RDG が使用） | 同 |

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.RHIThread.Enable` | `RHICommandList.cpp` |
| `r.D3D12.MaxCommandsPerCommandList` | `D3D12CommandList.cpp` |
| `r.D3D12.UseD3DDebugDevice` | `D3D12RHI.cpp` |
| `r.D3D12.EnableDRED` | `D3D12Adapter.cpp`（GPU クラッシュ診断） |
| `r.D3D12.GPUTimeout` | `D3D12Submission.cpp` |
| `r.RHI.RayTracing` | `RHI.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_rhi_commandlist]] | FRHICommandList 系 |
| Reference | [[Reference/ref_rhi_resources]] | FRHIResource 派生 |
| Reference | [[Reference/ref_rhi_dynamic_rhi]] | FDynamicRHI |
| Reference | [[Reference/ref_rhi_d3d12_device]] | FD3D12Device |
| Reference | [[Reference/ref_rhi_d3d12_context]] | FD3D12CommandContext |
| Reference | [[Reference/ref_rhi_d3d12_resources]] | FD3D12Texture/Buffer |
| Details | [[Details/a_rhi_abstraction]] | 抽象層の仕組み |
| Details | [[Details/b_rhi_resources]] | リソース参照カウント |
| Details | [[Details/c_rhi_d3d12_device]] | D3D12 Device 初期化 |
| Details | [[Details/d_rhi_d3d12_context]] | D3D12 コマンド記録 |
| Details | [[Details/e_rhi_rdg_bridge]] | RDG からの呼び出し境界 |

---

## ue5-dive 起点

- 「Draw が実際にどう GPU に届くか」 → `RHICommandList.cpp:Execute` → `D3D12Commands.cpp`
- 「バリアがいつ挿入されるか」 → `D3D12CommandList.cpp:FlushResourceBarriers`
- 「GPU クラッシュ診断」 → DRED（`r.D3D12.EnableDRED`）・`RHIBreadcrumbs.h`
- 「新しい RHI バックエンド追加」 → `FDynamicRHI` 継承 + `IRHICommandContext` 実装
