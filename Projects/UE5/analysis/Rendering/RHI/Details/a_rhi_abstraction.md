# RHI 抽象層

- 対象: `Runtime/RHI/Public/RHICommandList.h`, `RHIContext.h`, `DynamicRHI.h`
- 関連Reference: [[ref_rhi_commandlist]] | [[ref_rhi_dynamic_rhi]]

---

## 概要

RHI 抽象層は**プラットフォーム非依存のレンダリング API** を提供する。  
レンダラーはこの層のみと対話し、実際の GPU API（D3D12 / Vulkan / Metal）は知らなくてよい設計。

---

## 主要クラス構成

```
FDynamicRHI                         ← リソース生成・破棄の仮想インターフェース
  └── FD3D12DynamicRHI 等           ← プラットフォーム実装

IRHICommandContext                  ← コマンド記録の仮想インターフェース（コア）
  ├── IRHIComputeContext            ← Compute 系コマンド
  └── FD3D12ContextCommon 等        ← プラットフォーム実装

FRHICommandListBase                 ← コマンドリスト基底（メモリスタック管理）
  └── FRHICommandList               ← 遅延実行（マルチスレッド記録・後で Flush）
        └── FRHICommandListImmediate← 即時実行（メインレンダースレッドで使用）
              別名: GRHICommandList  ← グローバルアクセスポイント
```

---

## FDynamicRHI

`DynamicRHI.h` で定義。プラットフォーム RHI プラグインが継承し実装する。

### 役割
- GPU リソース（テクスチャ・バッファ・PSO 等）の **生成・破棄**
- スワップチェーン・ビューポート管理
- GPU クエリ・フェンス管理

### 主要メソッド（抜粋）

```cpp
// テクスチャ生成
virtual FTextureRHIRef RHICreateTexture(const FRHITextureCreateDesc& CreateDesc) = 0;

// バッファ生成
virtual FBufferRHIRef RHICreateBuffer(
    FRHICommandListBase& RHICmdList,
    const FRHIBufferCreateDesc& CreateDesc,
    ERHIAccess InResourceState) = 0;

// グラフィクス PSO 生成
virtual FGraphicsPipelineStateRHIRef RHICreateGraphicsPipelineState(
    const FGraphicsPipelineStateInitializer& Initializer) = 0;

// コンピュート PSO 生成
virtual FComputePipelineStateRHIRef RHICreateComputePipelineState(
    const FRHIComputeShader* ComputeShader) = 0;

// フレーム開始・終了
virtual void RHIBeginFrame() = 0;
virtual void RHIEndFrame(const FRHIEndFrameArgs& Args) = 0;
```

### グローバルアクセス

```cpp
// プロセス全体で1つ存在するシングルトン
extern RHI_API FDynamicRHI* GDynamicRHI;

// ショートカット関数（GDynamicRHI->RHICreateTexture と等価）
inline FTextureRHIRef RHICreateTexture(const FRHITextureCreateDesc& CreateDesc)
{
    return GDynamicRHI->RHICreateTexture(CreateDesc);
}
```

---

## IRHICommandContext

`RHIContext.h` で定義。コマンド記録（描画・コピー・バリア等）の純粋仮想インターフェース。

### 主要メソッド（抜粋）

```cpp
// パイプラインステート設定
virtual void RHISetGraphicsPipelineState(
    FRHIGraphicsPipelineState* GraphicsState,
    uint32 StencilRef,
    bool bApplyAdditionalState) = 0;

// 頂点バッファバインド
virtual void RHISetStreamSource(
    uint32 StreamIndex,
    FRHIBuffer* VertexBuffer,
    uint32 Offset) = 0;

// ドローコール
virtual void RHIDrawIndexedPrimitive(
    FRHIBuffer* IndexBuffer,
    int32 BaseVertexIndex,
    uint32 FirstInstance,
    uint32 NumVertices,
    uint32 StartIndex,
    uint32 NumPrimitives,
    uint32 NumInstances) = 0;

// コンピュートディスパッチ
virtual void RHIDispatchComputeShader(
    uint32 ThreadGroupCountX,
    uint32 ThreadGroupCountY,
    uint32 ThreadGroupCountZ) = 0;

// リソースバリア（トランジション）
virtual void RHISetTrackedAccess(const FRHICommandSetTrackedAccess& Args) = 0;

// レンダーターゲット設定
virtual void RHISetRenderTargets(
    uint32 NumSimultaneousRenderTargets,
    const FRHIRenderTargetView* NewRenderTargetsRHI,
    const FRHIDepthRenderTargetView* NewDepthStencilTargetRHI) = 0;
```

---

## FRHICommandList

`RHICommandList.h` で定義。RHI コマンドを**コマンドバッファに積んで遅延実行する**ための型。

### コマンド記録の仕組み

```
RHICmdList.DrawIndexedPrimitive(...)
    ↓
FRHICommandDrawIndexedPrimitive コマンド構造体を
メモリスタック（FMemStack）にアロケート
    ↓
コマンドリンクリストに追加
    ↓
Flush / Submit 時に IRHICommandContext へ送信
```

### FRHICommandListImmediate との違い

| | FRHICommandList | FRHICommandListImmediate |
|--|----------------|--------------------------|
| 実行タイミング | `Flush()` / `Execute()` 呼び出し時 | API 呼び出しと同時に（または RHI スレッドへ即 Enqueue）|
| 用途 | 並列コマンド記録（Task Graph 上） | メインレンダースレッドのコマンド記録 |
| スレッド | 任意のワーカースレッド | レンダースレッドのみ |

### グローバル変数

```cpp
// メインのレンダースレッド用即時コマンドリスト
extern RHI_API FRHICommandListImmediate GRHICommandList;
```

---

## コマンド記録フロー（概念）

```
[Render Thread]
  FRHICommandListImmediate& RHICmdList = GRHICommandList;

  RHICmdList.BeginRenderPass(...);         // コマンドを積む
  RHICmdList.SetGraphicsPipelineState(...);
  RHICmdList.DrawIndexedPrimitive(...);
  RHICmdList.EndRenderPass();

  ↓ RHI スレッドが非同期にデキュー

[RHI Thread]
  IRHICommandContext* Context = ...;       // D3D12ContextCommon 等
  Context->RHIBeginRenderPass(...);        // 仮想関数経由で D3D12 呼び出し
  Context->RHISetGraphicsPipelineState(...);
  Context->RHIDrawIndexedPrimitive(...);
  Context->RHIEndRenderPass();

  ↓ ExecuteCommandLists

[GPU]
  実際の描画実行
```

---

## 関連リファレンス

- [[ref_rhi_commandlist]] … FRHICommandListBase / FRHICommandList / FRHICommandListImmediate の詳細
- [[ref_rhi_dynamic_rhi]] … FDynamicRHI / IRHICommandContext のメソッド一覧
