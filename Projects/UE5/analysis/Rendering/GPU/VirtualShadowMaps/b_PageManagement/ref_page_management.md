# REF: VSM Page Management シェーダー

- グループ: b - Page Management
- 詳細: [[detail_page_management]]
- CPU リファレンス: [[ref_vsm_array]]
- ソース: `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapPageMarking.usf`
          `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapPageManagement.usf`
          `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapPhysicalPageManagement.usf`

---

## VirtualShadowMapPageMarking.usf

### PruneLightGridCS

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_X, 1, 1)]
void PruneLightGridCS(uint GridLinearIndex : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | Forward Light Grid から VSM 付きライトのみを抽出して `OutPrunedLightGridData` に書き込み、後続パスの処理量を削減 |
| **入力** | Light Grid データ, `MinLocalLightIndex`, `MaxLocalLightIndex`, `bIncludeMegaLights` |
| **出力** | `OutPrunedLightGridData`, `OutPrunedNumCulledLightsGrid` |
| **CPU 関数** | `FVirtualShadowMapArray::BeginMarkPages()` |

### GeneratePageFlagsFromPixels

```hlsl
[numthreads(SLW_TILE_SIZE_XY, SLW_TILE_SIZE_XY, 1)]
void GeneratePageFlagsFromPixels(
    uint3 InGroupId        : SV_GroupID,
    uint  GroupIndex       : SV_GroupIndex,
    uint3 GroupThreadId    : SV_GroupThreadID,
    uint3 DispatchThreadId : SV_DispatchThreadID
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | GBuffer / HairStrands / 水面の各ピクセルが影を受ける VSM ページに要求フラグを立てる |
| **パーミュテーション** | `PERMUTATION_INPUT_TYPE`: 0=GBuffer, 1=GBuffer+Water, 2=HairStrands |
| **入力** | `SceneDepth`, `GBufferNormal`, `HairStrands.HairOnlyDepthTexture`（HairStrands 時）|
| **出力** | `OutPageRequestFlags`（`MarkPageFroxel()` / `MarkPageClipmapFroxel()` 経由）|
| **CPU 関数** | `FVirtualShadowMapArray::BeginMarkPages()` |
| **注意** | First Person ピクセル（`bIsFirstPersonPixel`）は専用クリップマップのみをマーク |

### GeneratePageFlagsFromFroxelsCS

```hlsl
[numthreads(FROXEL_INDIRECT_ARG_WORKGROUP_SIZE, 1, 1)]
void GeneratePageFlagsFromFroxelsCS(uint FroxelIndex : SV_DispatchThreadID)
```

| 目的 | スポット/ポイントライトの Froxel（視錐台ボクセル）からページ要求を生成。半透明・フロントレイヤーを含む |
| 入力 | Froxel データ, ライトグリッド（`OutPrunedLightGridData`）|
| CPU 関数 | `FVirtualShadowMapArray::BeginMarkPages()` |

### MarkCoarsePages

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_X, 1, 1)]
void MarkCoarsePages(uint DispatchThreadId : SV_DispatchThreadID)
```

| 目的 | ディレクショナルライトの遠距離クリップマップページに `VSM_FLAG_COARSE_LEVEL` をセット |

---

## VirtualShadowMapPageManagement.usf

### GenerateHierarchicalPageFlags

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_X, 1, 1)]
void GenerateHierarchicalPageFlags(uint ThreadId : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 各物理ページのフラグを MIP 階層（H-flags）に伝播し、ページ矩形バウンドを計算 |
| **入力** | `PhysicalPageMetaData`（1スレッド = 1物理ページ）, `InPageFlags` |
| **出力** | `OutPageFlagMips[0..N]`（階層フラグ）, `OutAllocatedPageRectBounds`, `OutUncachedPageRectBounds` |
| **CPU 関数** | `FVirtualShadowMapArray::BuildPageAllocations()` |

### PropagateMappedMips

```hlsl
[numthreads(PER_PAGE_THREAD_GROUP_SIZE_XY, PER_PAGE_THREAD_GROUP_SIZE_XY, 1)]
void PropagateMappedMips(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 目的 | 割り当て済みページの情報を上位 MIP に伝播（`FMappedMipPropagator` パターン）|
| CPU 関数 | `FVirtualShadowMapArray::BuildPageAllocations()` |

---

## VirtualShadowMapPhysicalPageManagement.usf

### InitPageRectBounds

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_X, 1, 1)]
void InitPageRectBounds(uint3 Index : SV_DispatchThreadID)
```

| 目的 | ページ矩形バウンドバッファを初期値（INT_MAX/INT_MIN）でクリア（InterlockedMin/Max の事前準備）|

### UpdatePhysicalPageAddresses

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_X, 1, 1)]
void UpdatePhysicalPageAddresses(uint3 Index : SV_DispatchThreadID)
```

| 目的 | 物理ページの旧 → 新アドレスの変換テーブルを更新（ページ移動・再割り当て後の整合性維持）|

### UpdatePhysicalPages

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_X, 1, 1)]
void UpdatePhysicalPages(uint3 Index : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 要求フラグに基づき物理ページの割り当て／解放を決定。仮想ページテーブルを更新 |
| **入力** | `InPageFlags`, `PhysicalPageMetaData`, 空きページリスト |
| **出力** | 更新済み `PageTable`（仮想→物理マッピング）, 空きページリスト（変化分）|
| **CPU 関数** | `FVirtualShadowMapArray::BuildPageAllocations()` |

### ClearPageTableCS

```hlsl
[numthreads(PER_PAGE_THREAD_GROUP_SIZE_XY, PER_PAGE_THREAD_GROUP_SIZE_XY, 1)]
void ClearPageTableCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 目的 | ページテーブルの特定範囲をクリア（新規 VSM または再確保時）|

### AllocateNewPageMappingsCS

```hlsl
[numthreads(PER_PAGE_THREAD_GROUP_SIZE_XY, PER_PAGE_THREAD_GROUP_SIZE_XY, 1)]
void AllocateNewPageMappingsCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 目的 | 新規ページを物理プールから取得し、仮想ページテーブルに書き込む |
| 出力 | `PageTable`（新規エントリ追加）, `PhysicalPageMetaData`（初期化）|

### PackAvailablePages

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void PackAvailablePages(uint GroupIndex : SV_GroupIndex)
```

| 目的 | 空き物理ページリストを連続した配列にパック（断片化解消 / 次フレームの高速アロケーション用）|

### AppendPhysicalPageLists

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_X, 1, 1)]
void AppendPhysicalPageLists(uint ThreadId : SV_DispatchThreadID)
```

| 目的 | 解放された物理ページを空きページリストの末尾に追加 |

---

## 共有ヘッダー

| ファイル | 役割 |
|---------|------|
| `VirtualShadowMapPageAccessCommon.ush` | `CalcPageOffset()` / ページテーブルアクセス関数 |
| `VirtualShadowMapPageCacheCommon.ush` | `VSM_FLAG_*` 定数 / `FPhysicalPageMetaData` / キャッシュ判定ロジック |
| `VirtualShadowMapPageMarking.ush` | `MarkPageFroxel()` / `MarkPageClipmapFroxel()` インライン実装 |
| `VirtualShadowMapPageOverlap.ush` | ページ重複判定ユーティリティ |
| `VirtualShadowMapPerPageDispatch.ush` | `FPerPageDispatchSetup` — 1スレッド=1ページの処理パターン |

---

> [!note]- H-flags（階層ページフラグ）の目的
> VSM のページテーブルは MIP ピラミッド構造を持つ（MIP0 = 最高解像度）。  
> `GenerateHierarchicalPageFlags` が各ページのフラグを上位 MIP に伝播する「H-flags」を構築することで、  
> 描画時に「このエリアに有効なページが存在するか」を上位 MIP で高速に確認できる（早期終了が可能）。

> [!note]- キャッシュ済みページと未キャッシュページの分離
> `OutAllocatedPageRectBounds` には割り当て済みページ全体の矩形が入るが、  
> `OutUncachedPageRectBounds` には **このフレームに再描画が必要なページのみ** の矩形が入る。  
> 描画 Indirect Args は `OutUncachedPageRectBounds` を元に生成されるため、  
> 静的キャッシュページは描画フェーズで完全にスキップされる。

> [!note]- MarkPageFroxel と MarkPageClipmapFroxel の違い
> `MarkPageFroxel` はスポット/ポイントライトの単一 VSM（単一ビュー）にページをマークする。  
> `MarkPageClipmapFroxel` はディレクショナルライトのクリップマップで、ピクセルの深度に応じて  
> `GetBiasedClipmapLevel()` で適切なクリップマップレベルを選択してからマークを行う。  
> この分岐によりディレクショナルライトの解像度選択が LOD バイアスを含めて正確に行われる。
