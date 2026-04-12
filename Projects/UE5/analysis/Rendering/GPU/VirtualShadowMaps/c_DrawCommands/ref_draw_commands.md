# REF: VSM Draw Commands シェーダー

- グループ: c - Draw Commands
- 詳細: [[detail_draw_commands]]
- CPU リファレンス: [[ref_vsm_array]]
- ソース: `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapCompactViews.usf`
          `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapBuildPerPageDrawCommands.usf`
          `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapComputeExplicitChunkDrawsViewMask.usf`
          `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapShadowCasterBounds.usf`

---

## VirtualShadowMapCompactViews.usf

### CompactViewsVSM_CS

```hlsl
[numthreads(VSM_MAX_VIEWS_PER_GROUP, 1, 1)]
void CompactViewsVSM_CS(
    uint GroupId  : SV_GroupID,
    uint ThreadId : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | ビューグループを走査し、未キャッシュページが存在する（描画が必要な）ビューのみをコンパクト化した出力バッファに詰め込む |
| **入力** | `InOutViewDrawRanges`（ビューグループ定義）, `UncachedPageRectBounds`（ページ矩形バウンド）|
| **出力** | コンパクト化された `OutputCompactedViews`（描画対象ビューリスト）|
| **CPU 関数** | `FVirtualShadowMapArray::BuildPageAllocations()` |
| **特徴** | `ThreadId == 0` のみが処理（スカラーループ）— 各ビューグループのビュー数が小さいため十分高速 |

---

## VirtualShadowMapBuildPerPageDrawCommands.usf

### CullPerPageDrawCommandsCs

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void CullPerPageDrawCommandsCs(
    uint3 GroupId          : SV_GroupID,
    int GroupThreadIndex   : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | インスタンスカリングロードバランサーを使ってインスタンスを VSM ページ単位でカリングし、可視インスタンスリストと Indirect Draw Args を生成 |
| **入力** | `InstanceCullingLoadBalancer`（インスタンスバッチ）, `VSM ビュー定義`, `PageTable` |
| **出力** | `VisibleInstances`（FVSMVisibleInstanceCmd）, `DrawIndirectArgsBuffer`（インスタンス数）|
| **CPU 関数** | `FVirtualShadowMapArray::RenderVirtualShadowMapsNonNanite()` |
| **ENABLE_BATCH_MODE** | バッチモード時: `LoadBatchInfo(DispatchGroupId)` で複数コマンドを1 Dispatch で処理 |

### AllocateCommandInstanceOutputSpaceCs

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void AllocateCommandInstanceOutputSpaceCs(uint IndirectArgIndex : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 各 Draw Command のインスタンス数に基づいてインスタンス出力バッファの範囲を確保し、オフセットを書き込む |
| **入力** | `DrawIndirectArgsBuffer`（各コマンドのインスタンス数）|
| **出力** | `InstanceIdOffsetBufferOut`, `TmpInstanceIdOffsetBufferOut`（各コマンドの出力オフセット）, `OutputOffsetBufferOut`（累積合計）|
| **CPU 関数** | `FVirtualShadowMapArray::RenderVirtualShadowMapsNonNanite()` |

### OutputCommandInstanceListsCs

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void OutputCommandInstanceListsCs(uint VisibleInstanceIndex : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | `VisibleInstances` バッファから読み取り、インスタンスIDとページ情報を確保済みオフセット位置にScatter書き込み |
| **入力** | `VisibleInstances`（FVSMVisibleInstanceCmd）, `TmpInstanceIdOffsetBufferOut`（オフセット）|
| **出力** | `InstanceIdsBufferOut`（インスタンスIDリスト）, `PageInfoBufferOut`（ページ情報）|
| **CPU 関数** | `FVirtualShadowMapArray::RenderVirtualShadowMapsNonNanite()` |

---

## VirtualShadowMapComputeExplicitChunkDrawsViewMask.usf

### ComputeExplicitChunkDrawsViewMask

```hlsl
[numthreads(64, 1, 1)]
void ComputeExplicitChunkDrawsViewMask(
    uint3 GroupId           : SV_GroupID,
    uint GroupThreadIndex   : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Nanite Explicit チャンク Draw の `ActiveViewMask` を `NumViews` から計算して `InOutInstanceCullingWorkGroups` に書き込む |
| **入力** | `InViewDrawRanges`, `InOutInstanceCullingWorkGroups`, `NumWorkGroups` |
| **出力** | `InOutInstanceCullingWorkGroups`（`ActiveViewMask` 更新済み）|
| **CPU 関数** | `FVirtualShadowMapArray::BuildPageAllocations()` |

---

## VirtualShadowMapShadowCasterBounds.usf

### MainVS

```hlsl
void MainVS(
    float3 InPosition : ATTRIBUTE0,
    uint InstanceId   : SV_InstanceID,
    out FVSToPS Output
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | インスタンスの AABB 頂点を世界座標に変換して Shadow Depth パスのバウンドバッファに書き込む準備 |
| **入力** | `InPosition`（AABB コーナー座標）, `InstanceId`（インスタンスID）|
| **出力** | `Output.SvPosition`（クリップ空間座標）, `Output.CubePosition`, `Output.WorldPosition` |
| **注意** | 無効インスタンス（`!InstanceData.ValidInstance`）は `SvPosition = NaN` で HW カリング |

### MainPS

```hlsl
void MainPS(
    FVSToPS Input,
    out float4 OutColor : SV_Target0
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | インスタンスの AABB を深度バッファに書き込んでシャドウキャスターの有無を記録 |
| **出力** | `OutColor`（インスタンス情報エンコード）— HZB ベースカリングで参照 |

---

> [!note]- CompactViews が描画コストを削減する仕組み
> VSM は静的ジオメトリのページをキャッシュし、変化がなければ再描画しない。  
> `CompactViewsVSM_CS` で「`UncachedPageRectBounds` が有効なビュー」のみを取り出すことで、  
> 後続の Shadow Depth パス（Nanite/非 Nanite どちらも）は完全にスキップできるビューを事前に除外できる。  
> シーンが静的な部分が多いほどこの最適化の効果が大きい。

> [!note]- BuildPerPageDrawCommands の3フェーズ設計
> **フェーズ1 (CullPerPageDrawCommands)** でインスタンス数を数えるだけにとどめ、  
> **フェーズ2 (AllocateCommandInstanceOutputSpace)** でバッファ範囲を確保してから  
> **フェーズ3 (OutputCommandInstanceLists)** でScatter書き込みを行う。  
> この3フェーズはNaniteのRasterBinBuildと同様のプレフィックスサム相当のパターンで、  
> GPU上でのダイナミックバッファアロケーションの標準的なイディオム。

> [!note]- FVSMVisibleInstanceCmd の PageInfo の役割
> `CullPerPageDrawCommandsCs` が出力する `FVSMVisibleInstanceCmd` には  
> インスタンスIDだけでなく「どのページに影響するか」の情報（PageInfo）が含まれる。  
> `OutputCommandInstanceListsCs` がこれを `PageInfoBufferOut` に書き出し、  
> Shadow Depth シェーダーが「自分がどのページ座標に描画すべきか」を知るために使用する。  
> これにより1インスタンスが複数ページにまたがる場合でも正確な座標変換が可能。
