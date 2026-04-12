# VSM Draw Commands シェーダー詳細

- グループ: c - Draw Commands
- GPU 概要: [[01_vsm_gpu_overview]]
- CPU 詳細: [[a_vsm_array]]
- リファレンス: [[ref_draw_commands]]

---

## 概要

VSM のシャドウ描画準備パイプライン。物理ページが確定した後、  
実際にシャドウを描画するための **ビューのコンパクト化 → インスタンスカリング → Draw Command 生成** を行う。

```
CompactViews（VirtualShadowMapCompactViews.usf）
  → 描画が必要な VSM ビュー（Uncached ページがあるもの）を絞り込み

BuildPerPageDrawCommands（VirtualShadowMapBuildPerPageDrawCommands.usf）
  ├─ CullPerPageDrawCommandsCs … インスタンスをページ単位でカリング
  ├─ AllocateCommandInstanceOutputSpaceCs … Draw Command バッファを確保
  └─ OutputCommandInstanceListsCs … インスタンスリストを最終バッファに出力

ComputeExplicitChunkDrawsViewMask（VirtualShadowMapComputeExplicitChunkDrawsViewMask.usf）
  → Explicit チャンク Draw のビューマスクを計算

ShadowCasterBounds（VirtualShadowMapShadowCasterBounds.usf）
  → インスタンスのバウンディングボックスを Shadow Depth バッファに書き込む（カリング用）
```

---

## CompactViewsVSM_CS のコアロジック

```hlsl
[numthreads(VSM_MAX_VIEWS_PER_GROUP, 1, 1)]
void CompactViewsVSM_CS(uint GroupId : SV_GroupID, uint ThreadId : SV_GroupIndex)
```

`InOutViewDrawRanges` の各ビューグループを走査し、  
`UncachedPageRectBounds` が有効な（描画が必要な）ビューのみを  
コンパクトした出力バッファに詰め込む。

```hlsl
// ThreadId=0 のみが処理（スカラー実行 → Wave Ops は利用せず汎用性を重視）
if (ThreadId == 0)
{
    FViewDrawGroup ViewDrawGroup = InOutViewDrawRanges[ViewGroupIndex];

    for each PrimaryView in ViewDrawGroup:
        for each MipLevel:
            uint4 RectPages = UncachedPageRectBounds[...];
            // 矩形バウンドが有効（zw >= xy）→ 描画が必要
            if (all(RectPages.zw >= RectPages.xy))
            {
                // このビュー×MIP を出力バッファにコピー
                OutputCompactedViews[OutputViewOffset++] = CurrentView;
            }
}
```

これにより後続の Shadow Depth パスは未変更のキャッシュ済みページを持つビューを完全にスキップできる。

---

## BuildPerPageDrawCommands のコアロジック

### フェーズ1: CullPerPageDrawCommandsCs

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void CullPerPageDrawCommandsCs(uint3 GroupId : SV_GroupID, int GroupThreadIndex : SV_GroupIndex)
```

`InstanceCullingLoadBalancer` を使ってインスタンスバッチを処理：
1. インスタンスの AABB を VSM ビューで変換
2. フラスタムカリング（ビュー外なら除外）
3. ページ単位のカリング（このインスタンスが影響するページを特定）
4. 可視インスタンスを `VisibleInstances` バッファに Append
5. 対応する Indirect Draw Args をインクリメント（`DrawIndirectArgsBuffer`）

### フェーズ2: AllocateCommandInstanceOutputSpaceCs

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void AllocateCommandInstanceOutputSpaceCs(uint IndirectArgIndex : SV_DispatchThreadID)
```

フェーズ1で計算した各 Draw Command のインスタンス数を元に、  
`OutputOffsetBuffer` にバッファ範囲を確保（`InterlockedAdd`）。  
同時に `InstanceIdOffsetBufferOut` と `TmpInstanceIdOffsetBufferOut` に  
各 Draw Command のインスタンス配置オフセットを書き込む。

### フェーズ3: OutputCommandInstanceListsCs

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void OutputCommandInstanceListsCs(uint VisibleInstanceIndex : SV_DispatchThreadID)
```

`VisibleInstances` バッファから読み取り、インスタンスIDとページ情報を  
対応する Draw Command の出力バッファ（`InstanceIdsBufferOut`, `PageInfoBufferOut`）に  
確保済みオフセットに Scatter 書き込みする。

```hlsl
uint InstanceIdOutputOffset;
InterlockedAdd(TmpInstanceIdOffsetBufferOut[VisibleInstanceCmd.IndirectArgIndex],
               1U, InstanceIdOutputOffset);
InstanceIdsBufferOut[...] = InstanceId;
PageInfoBufferOut[...]     = PageInfo;
```

---

## ShadowCasterBounds のコアロジック

```hlsl
void MainVS(float3 InPosition : ATTRIBUTE0, uint InstanceId : SV_InstanceID,
            out FVSToPS Output)
void MainPS(FVSToPS Input, out float4 OutColor : SV_Target0)
```

インスタンスのバウンディングボックスを VSM の深度バッファに描画する。  
これにより、Shadow Depth パスで使う HZB ベースのカリングに使用できる深度情報を事前に書き込む。  
無効なインスタンス（`!InstanceData.ValidInstance`）は `SvPosition = NaN` で除外。

---

## ComputeExplicitChunkDrawsViewMask のコアロジック

```hlsl
[numthreads(64, 1, 1)]
void ComputeExplicitChunkDrawsViewMask(uint3 GroupId, uint GroupThreadIndex)
```

`FInstanceCullingGroupWork` の各エントリに対して  
`ViewDrawGroup.NumViews` から全ビューマスクを計算して書き込む。  
Nanite の明示的なチャンク描画がすべてのビューを処理するために必要。

---

## 入出力

| リソース | 内容 |
|---------|------|
| `InOutViewDrawRanges` | ビューグループの定義（CompactViews 前後で使用）|
| `UncachedPageRectBounds` | 未キャッシュページの矩形バウンド（CompactViews の判断材料）|
| `CompactedViews` | 描画が必要なビューのコンパクトリスト（出力）|
| `VisibleInstances` | ページカリング後の可視インスタンスリスト（一時バッファ）|
| `DrawIndirectArgsBuffer` | Draw Command の Indirect Args（インスタンス数）|
| `InstanceIdsBufferOut` | 最終的なインスタンス ID リスト（Shadow Depth パスが参照）|
| `PageInfoBufferOut` | 各インスタンスが影響するページ情報 |

---

## CPU 呼び出しの流れ

```
FVirtualShadowMapArray::BuildPageAllocations()    // 後半
  └─ CompactViewsVSM_CS

FVirtualShadowMapArray::RenderVirtualShadowMapsNonNanite()
  ├─ CullPerPageDrawCommandsCs（Indirect Dispatch）
  ├─ AllocateCommandInstanceOutputSpaceCs
  ├─ OutputCommandInstanceListsCs（Indirect Dispatch）
  └─ [Shadow Depth パスで DrawIndirectArgsBuffer を使って描画]

FVirtualShadowMapArray::BuildPageAllocations() 内
  └─ ComputeExplicitChunkDrawsViewMask（Nanite パス用）
```
