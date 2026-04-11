# RDG ↔ RHI ブリッジ

- 対象: `Runtime/RenderCore/Public/RenderGraphBuilder.h`, `RenderGraphPass.h`, `RenderGraphResources.h`
- 関連: [[10_rdg_overview]] | [[ref_rhi_commandlist]] | [[ref_rhi_d3d12_context]]

---

## 概要

**RDG（Render Dependency Graph）は RHI の上に乗る高レベル抽象層**。  
RDG は「どのリソースをどの順で使うか」を宣言的に記述し、`Execute()` 時に:
1. バリア（リソーストランジション）を自動挿入
2. 使われないパスを省略（カリング）
3. `FRDGTexture` 等の仮想リソースを実 `FRHITexture*` に確定
4. 各パスのラムダを `FRHICommandList` 越しに実行

---

## RDG と RHI のリソース対応

| RDG リソース | 実 RHI リソース | 説明 |
|------------|----------------|------|
| `FRDGTexture*` | `FRHITexture*` | テクスチャ。Execute 時に実体確定 |
| `FRDGBuffer*` | `FRHIBuffer*` | バッファ。Execute 時に実体確定 |
| `FRDGTextureRef` | `FRHITexture*` | `FRDGTexture*` の別名（ typedef ）|
| `FRDGBufferRef` | `FRHIBuffer*` | `FRDGBuffer*` の別名 |
| `FRDGTextureSRV` | `FRHIShaderResourceView*` | テクスチャ SRV |
| `FRDGTextureUAV` | `FRHIUnorderedAccessView*` | テクスチャ UAV |
| `FRDGBufferSRV` | `FRHIShaderResourceView*` | バッファ SRV |
| `FRDGBufferUAV` | `FRHIUnorderedAccessView*` | バッファ UAV |

実 RHI リソースへのアクセスはパスのラムダ内で `PassParameters->Texture->GetRHI()` のように行う。

---

## RDG パス登録 → RHI 実行の流れ

```
─── Build フェーズ ────────────────────────────────────────────
[Render Thread]

GraphBuilder.CreateTexture(Desc, TEXT("MyTex"))
  → FRDGTexture* を返す（まだ FRHITexture は存在しない）

GraphBuilder.AddPass(
  RDG_EVENT_NAME("MyPass"),
  PassParameters,
  ERDGPassFlags::Compute,
  [PassParameters](FRHIComputeCommandList& RHICmdList)
  {
    // このラムダは Execute 時に呼ばれる
    FRHITexture* RHITex = PassParameters->MyTexture->GetRHI();
    RHICmdList.SetComputeShader(...);
    RHICmdList.DispatchComputeShader(...);
  });

─── Execute フェーズ ──────────────────────────────────────────
GraphBuilder.Execute()
  │
  ├─ [1] グラフ解析
  │     FRDGPass の依存関係からバリアを計算
  │     使われていないパスをカリング
  │
  ├─ [2] リソース割り当て
  │     FRDGTexture ごとに FRHITexture* をアロケート
  │     Transient リソースはメモリエイリアシングで節約
  │
  ├─ [3] パスを順番に実行
  │     各 FRDGPass::Execute():
  │       ├─ 前バリア: ERHIAccess の変換を FD3D12ContextCommon::AddPendingBarrier()
  │       ├─ パスラムダを呼び出し（FRHICommandList を渡す）
  │       └─ 後バリア: 次パスへの遷移
  │
  └─ [4] リソース後処理
        Transient リソースを解放
        ExtractedResource を外部 TRefCountPtr に書き出し
```

---

## バリア自動生成の仕組み

```
パス A: FRDGTexture を UAV として書き込む (ERHIAccess::UAVCompute)
パス B: FRDGTexture を SRV として読み取る (ERHIAccess::SRVGraphics)

→ RDG は A→B 間に以下のバリアを自動挿入:
   D3D12_RESOURCE_BARRIER { Type = TRANSITION,
     StateBefore = D3D12_RESOURCE_STATE_UNORDERED_ACCESS,
     StateAfter  = D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE }
```

RDG が `ERHIAccess` → `D3D12_RESOURCE_STATES` の変換を行い、  
`FD3D12ContextCommon::FlushResourceBarriers()` でまとめて D3D12 API に送られる。

---

## パスラムダ内での RHI 使用パターン

```cpp
// Compute パスの例
FMyPassParameters* PassParameters = GraphBuilder.AllocParameters<FMyPassParameters>();
PassParameters->InputTexture  = GraphBuilder.CreateSRV(MyRDGTexture);
PassParameters->OutputBuffer  = GraphBuilder.CreateUAV(MyRDGBuffer);
PassParameters->View          = View.ViewUniformBuffer;

GraphBuilder.AddPass(
    RDG_EVENT_NAME("MyComputePass"),
    PassParameters,
    ERDGPassFlags::Compute,
    [PassParameters, ComputeShader](FRHIComputeCommandList& RHICmdList)
    {
        // RDG リソースから実 RHI リソースを取得
        FRHIShaderResourceView*   SRV = PassParameters->InputTexture->GetRHI();
        FRHIUnorderedAccessView*  UAV = PassParameters->OutputBuffer->GetRHI();

        // RHI コマンドを直接呼ぶ
        SetComputePipelineState(RHICmdList, ComputeShader);
        SetShaderUAV(RHICmdList, ComputeShader, UAV);
        SetShaderSRV(RHICmdList, ComputeShader, SRV);
        RHICmdList.DispatchComputeShader(GroupX, GroupY, 1);
    });
```

---

## Transient リソースとメモリエイリアシング

```
Pass A: Write → RDGTextureA (Transient)
Pass B: Read  ← RDGTextureA, Write → RDGTextureB (Transient)
Pass C: Read  ← RDGTextureB

→ RDGTextureA の生存期間は Pass A〜B のみ
   Pass C の後は RDGTextureA のメモリを解放
   RDGTextureB が同じメモリを使える（エイリアシング）
```

`ERDGPassFlags::NeverCull` または `GraphBuilder.ConvertToExternalTexture()` で  
リソースをグラフ外部に引き出すと Transient 扱いにならない。

---

## RDG からの RHI リソース取得

```cpp
// RDG グラフ実行後に実 RHI リソースを取り出す
FRDGTexture* RDGTex = GraphBuilder.CreateTexture(Desc, TEXT("Output"));
GraphBuilder.AddPass(...); // 書き込みパス

// 外部に引き出す（Execute 後も有効なポインタ）
TRefCountPtr<IPooledRenderTarget> PooledRT;
GraphBuilder.QueueTextureExtraction(RDGTex, &PooledRT);

GraphBuilder.Execute();

// 実行後は PooledRT->GetRenderTargetItem().TargetableTexture で FRHITexture* にアクセス
```

---

## 関連ドキュメント

- [[10_rdg_overview]] … RDG 全体の詳細説明
- [[a_rdg_builder]] … FRDGBuilder の詳細
- [[ref_rhi_commandlist]] … FRHICommandList の API 詳細
- [[ref_rhi_d3d12_context]] … D3D12 コンテキストの実装詳細
