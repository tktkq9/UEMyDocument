# VSM Page Management シェーダー詳細

- グループ: b - Page Management
- GPU 概要: [[01_vsm_gpu_overview]]
- CPU 詳細: [[a_vsm_array]]
- リファレンス: [[ref_page_management]]

---

## 概要

VSM のページ管理パイプライン。3つのシェーダーファイルが連携して  
「ページ要求フラグの生成 → 階層的伝播 → 物理ページの割り当て」を実行する。

```
Page Marking（VirtualShadowMapPageMarking.usf）
  → どのページが必要か（ページ要求フラグ）を生成

Page Management（VirtualShadowMapPageManagement.usf）
  → ページ要求フラグを MIP 階層に伝播
  → ページの矩形バウンドを計算

Physical Page Management（VirtualShadowMapPhysicalPageManagement.usf）
  → 物理ページアトラスへの割り当て
  → 仮想ページテーブルの更新
```

---

## Page Marking のコアロジック

### GeneratePageFlagsFromPixels — GBuffer からページ要求生成

```hlsl
[numthreads(SLW_TILE_SIZE_XY, SLW_TILE_SIZE_XY, 1)]
void GeneratePageFlagsFromPixels(
    uint3 InGroupId          : SV_GroupID,
    uint  GroupIndex         : SV_GroupIndex,
    uint3 GroupThreadId      : SV_GroupThreadID,
    uint3 DispatchThreadId   : SV_DispatchThreadID
)
```

各ピクセルの深度・法線を GBuffer から読み取り、  
そのピクセルが影を受ける VSM ページをマークする。

```
1. GBuffer から DeviceZ・WorldNormal を読み取り
2. 有効ピクセルを確認（背景・First Person ピクセルは特別処理）
3. 各 VSM についてピクセルの世界座標をページ座標に変換
4. `MarkPageFroxel()` または `MarkPageClipmapFroxel()` を呼び出して
   OutPageRequestFlags に VSM_FLAG_ALLOCATED | VSM_FLAG_DETAIL_GEOMETRY を書き込み
```

`PERMUTATION_INPUT_TYPE` で入力タイプを切り替え：
- `INPUT_TYPE_GBUFFER` — 標準 GBuffer（不透明オブジェクト）
- `INPUT_TYPE_GBUFFER_AND_WATER_DEPTH` — 水面深度も含む
- `INPUT_TYPE_HAIRSTRANDS` — Hair Strands 専用タイルを参照

### PruneLightGridCS — 不要ライトの除去

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_X, 1, 1)]
void PruneLightGridCS(uint GridLinearIndex : SV_DispatchThreadID)
```

Forward Light Grid から VSM を持つライトのみを抽出して `OutPrunedLightGridData` に書き込む。  
後続の `GeneratePageFlagsFromFroxelsCS` で参照するリストを小さくしてパフォーマンスを改善する。

### GeneratePageFlagsFromFroxelsCS — Froxel からページ要求

```hlsl
[numthreads(FROXEL_INDIRECT_ARG_WORKGROUP_SIZE, 1, 1)]
void GeneratePageFlagsFromFroxelsCS(uint FroxelIndex : SV_DispatchThreadID)
```

スポット/ポイントライトの視錐台ボクセル（Froxel）単位でページ要求フラグを生成。  
半透明オブジェクトやフロントレイヤーが影を受けるページもここでマークされる。

### MarkCoarsePages — ディレクショナルライト粗ページ

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_X, 1, 1)]
void MarkCoarsePages(uint DispatchThreadId : SV_DispatchThreadID)
```

ディレクショナルライトのクリップマップで、遠距離（粗解像度 MIP）のページに  
`VSM_FLAG_COARSE_LEVEL` フラグを立てる。近距離の詳細ページとは独立して処理する。

---

## Page Management のコアロジック

### GenerateHierarchicalPageFlags — MIP 階層への伝播

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_X, 1, 1)]
void GenerateHierarchicalPageFlags(uint ThreadId : SV_DispatchThreadID)
```

1スレッドが1物理ページを処理。ページのフラグを読み取り、  
親 MIP レベル（階層 H-flags）に伝播させてページバウンド矩形を計算する。

```hlsl
// ページが有効 → 親 MIP に伝播
for (uint HLevel = 0; HLevel < MaxHLevel; HLevel++)
{
    // ProcessMipLevel() で上位 MIP の対応するエントリに OR でフラグをセット
    bool bDone = ProcessMipLevel(OutPageFlagMips[HLevel], Flag, ...);
    if (bDone) break;
}

// 描画に必要なページ範囲（矩形バウンド）を InterlockedMin/Max で計算
InterlockedMin/Max(OutAllocatedPageRectBounds[...]);
InterlockedMin/Max(OutUncachedPageRectBounds[...]);  // キャッシュなし分のみ
```

### PropagateMappedMips — 割り当て済みページの上位 MIP への伝播

```hlsl
[numthreads(PER_PAGE_THREAD_GROUP_SIZE_XY, PER_PAGE_THREAD_GROUP_SIZE_XY, 1)]
void PropagateMappedMips(uint3 DispatchThreadId : SV_DispatchThreadID)
```

`FPerPageDispatchSetup` + `FMappedMipPropagator` パターンで、  
1スレッドが1ページを担当して上位 MIP への伝播処理を行う。

---

## Physical Page Management のコアロジック

### UpdatePhysicalPages — 物理ページ割り当て・解放

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_X, 1, 1)]
void UpdatePhysicalPages(uint3 Index : SV_DispatchThreadID)
```

最も処理量が多いカーネル。ページフラグを元に物理ページの割り当てと解放を決定する：

```
1. ページが要求済みかを確認（InPageFlags）
2. 既存の物理ページがキャッシュ有効か確認
3. 新規割り当てが必要 → AllocateNewPageMappings() でアドレスを確保
4. 不要ページ → PhysicalPageList（空きリスト）に返却
5. 仮想ページテーブルのエントリを更新
```

### AllocateNewPageMappingsCS — 新規ページマッピング確保

```hlsl
[numthreads(PER_PAGE_THREAD_GROUP_SIZE_XY, PER_PAGE_THREAD_GROUP_SIZE_XY, 1)]
void AllocateNewPageMappingsCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

新しいページを物理プールから割り当てて仮想ページテーブルに書き込む。  
`AllocateNewPageMappings()` インライン関数が実際の割り当てロジックを担当。

### PackAvailablePages + AppendPhysicalPageLists

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void PackAvailablePages(uint GroupIndex : SV_GroupIndex)

[numthreads(VSM_DEFAULT_CS_GROUP_X, 1, 1)]
void AppendPhysicalPageLists(uint ThreadId : SV_DispatchThreadID)
```

- `PackAvailablePages`: 空き物理ページリストを連続バッファにパック（断片化解消）
- `AppendPhysicalPageLists`: 解放されたページを空きリストに戻す

---

## 入出力

| リソース | 内容 |
|---------|------|
| `InPageFlags` | ページマーキング結果（要求フラグビット）|
| `OutPageRequestFlags` | GeneratePageFlagsFromPixels が書き込む要求フラグ |
| `OutPageFlagMips[0..N]` | 階層的ページフラグ MIP テクスチャ配列 |
| `OutAllocatedPageRectBounds` | 割り当て済みページ範囲（描画 Indirect Args の入力）|
| `OutUncachedPageRectBounds` | キャッシュなしページ範囲（再描画が必要な領域）|
| `PageTable` | 仮想ページ → 物理ページアドレスのマッピングテーブル |
| `PhysicalPagePool` | 物理ページアトラス（深度テクスチャ配列）|
| `PhysicalPageMetaData` | 各物理ページのメタデータ（VSM ID, MIP, ページ座標）|

---

## CPU 呼び出しの流れ

```
FVirtualShadowMapArray::BeginMarkPages()        // VirtualShadowMapArray.cpp:2106
  │
  ├─ PruneLightGridCS
  ├─ GeneratePageFlagsFromPixels（GBuffer, HairStrands, Water）
  ├─ GeneratePageFlagsFromFroxelsCS
  └─ MarkCoarsePages

FVirtualShadowMapArray::BuildPageAllocations()  // VirtualShadowMapArray.cpp:2818
  │
  ├─ GenerateHierarchicalPageFlags
  ├─ PropagateMappedMips
  ├─ InitPageRectBounds
  ├─ UpdatePhysicalPageAddresses
  ├─ UpdatePhysicalPages
  ├─ ClearPageTableCS
  ├─ AllocateNewPageMappingsCS
  ├─ PackAvailablePages
  └─ AppendPhysicalPageLists
```
