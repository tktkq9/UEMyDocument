# リファレンス：LumenSurfaceCache.cpp

- グループ: a - Surface Cache
- 上位: [[a_lumen_surface_cache]]
- 関連: [[ref_lumen_scene_data]] | [[ref_lumen_mesh_cards]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSurfaceCache.cpp`

---

## 概要

Surface Cache の**物理ページアロケーション・アトラス管理・更新スケジューリング**を実装するファイル。  
`FLumenSurfaceCacheAllocator` を使って Card のアトラス領域を割り当て・解放し、  
どの Card を今フレームで更新するかを決定する。

---

## 主要処理フロー

```
毎フレーム（UpdateSurfaceCacheAllocationState → BuildCardUpdateList → CaptureCards）:

  1. フィードバック読み取り（GPU → CPU）
     └─ FLumenSurfaceCacheFeedback::GetLatestReadbackBuffer() でページ参照頻度を取得

  2. 解像度決定（ResLevel Allocation）
     ├─ カメラ距離 → DesiredLockedResLevel を計算
     └─ アトラス空き容量 → 実際の ResLevel を決定（足りなければ格下げ）

  3. ページアロケーション
     ├─ FLumenSurfaceCacheAllocator::Allocate() で物理ページを確保
     └─ FLumenPageTableEntry に PhysicalPageCoord / PhysicalAtlasRect を書き込む

  4. 更新対象の選定（CardCaptureRequest リスト作成）
     ├─ 新規に割り当てられたページ → 優先してキャプチャ
     ├─ フィードバックで参照頻度が高いページ → 高解像度割り当て
     └─ r.LumenScene.SurfaceCache.CardCaptureRefreshFraction 分の既存ページ再キャプチャ

  5. ページ解放
     └─ 参照されなくなった Card のページを FLumenSurfaceCacheAllocator::Free() で解放
```

---

## UpdateSurfaceCacheAllocationState

```cpp
void UpdateSurfaceCacheAllocationState(
    FLumenSceneData& LumenData,
    const TArray<FViewInfo>& Views,
    FRHIGPUMask GPUMask,
    TArray<FSurfaceCacheRequest, SceneRenderingAllocator>& OutSurfaceCacheRequests);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `LumenData` | `FLumenSceneData&` | Lumen シーンデータ（Card の解像度とページ状態を更新）|
| `Views` | `const TArray<FViewInfo>&` | ビュー一覧（カメラ距離計算に使用）|
| `GPUMask` | `FRHIGPUMask` | マルチ GPU マスク |
| `OutSurfaceCacheRequests` | `TArray<FSurfaceCacheRequest>&` | 出力: キャプチャが必要なページのリクエスト |

### 使用箇所
- [[ref_lumen_scene]] `FDeferredShadingRenderer::UpdateLumenScene()` — GPU バッファ更新前に呼ばれる

### 内部処理フロー

1. **デバッグ/リセット処理**
   ```cpp
   if (CVarLumenSurfaceCacheReset.GetValueOnRenderThread()) {
       // 全カードのページを解放してリセット
       for (FLumenCard& Card : LumenData.Cards) {
           FreeSurfaceCacheAllocation(Card, LumenData.SurfaceCacheAllocator);
       }
   }
   ```

2. **GPU フィードバックの読み取り**
   ```cpp
   const FLumenSurfaceCacheFeedback::FReadbackBuffer* ReadbackBuffer =
       LumenData.SurfaceCacheFeedback.GetLatestReadbackBuffer();
   if (ReadbackBuffer) {
       LumenData.UpdateSurfaceCacheFeedback(*ReadbackBuffer, ...);
   }
   ```

3. **各 Card の解像度決定**
   ```cpp
   for (int32 CardIndex = 0; CardIndex < LumenData.Cards.Num(); CardIndex++) {
       FLumenCard& Card = LumenData.Cards[CardIndex];
       // カメラ距離からDesiredLockedResLevelを計算
       float DistanceSq = ComputeCardDistanceSqToNearestCamera(Card, Views);
       Card.DesiredLockedResLevel = ComputeDesiredResLevel(DistanceSq, Card.ResolutionScale);
   }
   ```

4. **ページ確保 / 格下げ**
   ```cpp
   for (FLumenCard& Card : LumenData.Cards) {
       int32 TargetResLevel = Card.DesiredLockedResLevel;
       // 空き不足なら格下げ
       while (!LumenData.SurfaceCacheAllocator.IsSpaceAvailable(Card, TargetResLevel, false)) {
           TargetResLevel--;
           if (TargetResLevel < Lumen::MinResLevel) break;
       }
       if (Card.MaxAllocatedResLevel < TargetResLevel) {
           LumenData.ReallocVirtualSurface(Card, CardIndex, TargetResLevel, true);
           // キャプチャリクエストを積む
           OutSurfaceCacheRequests.Add({CardIndex, TargetResLevel, ...});
       }
   }
   ```

5. **LRU eviction**（アトラスが満杯の場合）
   ```cpp
   while (!LumenData.SurfaceCacheAllocator.IsSpaceAvailable(...)) {
       TSparseUniqueList<int32, SceneRenderingAllocator> DirtyCards;
       LumenData.EvictOldestAllocation(MaxFramesSinceLastUsed, DirtyCards);
       // dirty になった Card は再キャプチャをスケジュール
   }
   ```

---

## BuildCardUpdateList

```cpp
void BuildCardUpdateList(
    FLumenSceneData& LumenData,
    const TArray<FViewInfo>& Views,
    TArray<FSurfaceCacheRequest, SceneRenderingAllocator>& SurfaceCacheRequests,
    FLumenCardRenderer& OutLumenCardRenderer);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `LumenData` | `FLumenSceneData&` | Lumen シーンデータ |
| `Views` | `const TArray<FViewInfo>&` | ビュー一覧 |
| `SurfaceCacheRequests` | `TArray<FSurfaceCacheRequest>&` | `UpdateSurfaceCacheAllocationState()` で生成されたリクエスト |
| `OutLumenCardRenderer` | `FLumenCardRenderer&` | 出力: 実際にキャプチャする Card ページリストを書き込む |

### 使用箇所
- [[ref_lumen_scene]] `FDeferredShadingRenderer::UpdateLumenScene()` — アロケーション更新直後に呼ばれる

### 内部処理フロー

1. **リクエストを優先度でソート**
   ```cpp
   SurfaceCacheRequests.Sort([](const FSurfaceCacheRequest& A, const FSurfaceCacheRequest& B) {
       // 新規割り当てページを先に、次に距離が近い順
       return A.Priority > B.Priority;
   });
   ```

2. **枚数・テクセル数の上限でクリップ**
   ```cpp
   const int32 MaxCardCapturesPerFrame = CVarLumenSurfaceCacheCardCapturesPerFrame.GetValueOnRenderThread(); // 300
   const int32 MaxTexels = TotalAtlasTexels / CVarLumenSurfaceCacheCardCaptureFactor.GetValueOnRenderThread(); // /64

   int32 TotalTexels = 0;
   for (int32 i = 0; i < SurfaceCacheRequests.Num(); i++) {
       TotalTexels += GetRequestTexelCount(SurfaceCacheRequests[i]);
       if (i >= MaxCardCapturesPerFrame || TotalTexels > MaxTexels) {
           SurfaceCacheRequests.SetNum(i); // 超えた分を切る
           break;
       }
   }
   ```

3. **再キャプチャ分のバジェット確保**
   ```cpp
   // r.LumenScene.SurfaceCache.CardCaptureRefreshFraction 割合分の既存ページも追加
   int32 RefreshBudget = MaxCardCapturesPerFrame * CardCaptureRefreshFraction;
   AddRefreshRequests(LumenData, Views, RefreshBudget, SurfaceCacheRequests);
   ```

4. **LumenCardRenderer にリストを渡す**
   ```cpp
   OutLumenCardRenderer.CardPagesToRender = SurfaceCacheRequests;
   ```

---

## AllocateSurfaceCacheAtlas

```cpp
void FLumenSceneData::AllocateCardAtlases(
    FRDGBuilder& GraphBuilder,
    FLumenSceneFrameTemporaries& FrameTemporaries,
    const FSceneViewFamily* ViewFamily);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `FrameTemporaries` | `FLumenSceneFrameTemporaries&` | 出力: 各アトラスの RDG テクスチャ参照 |
| `ViewFamily` | `const FSceneViewFamily*` | 圧縮設定・プラットフォーム判定 |

### 使用箇所
- [[ref_lumen_scene]] `FDeferredShadingRenderer::UpdateLumenScene()` の冒頭

### 内部処理フロー

1. **物理アトラスサイズの決定**
   ```cpp
   const int32 AtlasSize = CVarLumenSurfaceCacheAtlasSize.GetValueOnRenderThread(); // 4096
   const FIntPoint PhysicalAtlasSize(AtlasSize, AtlasSize);
   ```

2. **圧縮方式の選択**
   ```cpp
   ESurfaceCacheCompression Compression = GetSurfaceCacheCompression();
   EPixelFormat AlbedoFormat  = (Compression != Disabled) ? PF_BC7   : PF_R8G8B8A8;
   EPixelFormat NormalFormat  = (Compression != Disabled) ? PF_BC5   : PF_R8G8;
   EPixelFormat EmissiveFormat= (Compression != Disabled) ? PF_BC6H  : PF_FloatR11G11B10;
   ```

3. **Surface Cache アトラス（永続）の生成**
   ```cpp
   if (bReallocateAtlas || !AlbedoAtlas.IsValid()) {
       AlbedoAtlas   = RHICreateTexture2D(AtlasSize, AtlasSize, AlbedoFormat, 1, 1, TexCreate_ShaderResource | TexCreate_UAV, ...);
       OpacityAtlas  = RHICreateTexture2D(AtlasSize, AtlasSize, PF_G8, ...);
       DepthAtlas    = RHICreateTexture2D(AtlasSize, AtlasSize, PF_G16, ...);
       NormalAtlas   = RHICreateTexture2D(AtlasSize, AtlasSize, NormalFormat, ...);
       EmissiveAtlas = RHICreateTexture2D(AtlasSize, AtlasSize, EmissiveFormat, ...);
   }
   ```

4. **ライティングアトラスの生成**
   ```cpp
   DirectLightingAtlas   = RHICreateTexture2D(AtlasSize, AtlasSize, PF_FloatRGBA, ...);
   IndirectLightingAtlas = RHICreateTexture2D(AtlasSize, AtlasSize, PF_FloatRGBA, ...);
   FinalLightingAtlas    = RHICreateTexture2D(AtlasSize, AtlasSize, PF_FloatRGBA, ...);
   ```

5. **RDG へ登録して FrameTemporaries に書き込む**
   ```cpp
   FrameTemporaries.AlbedoAtlas   = GraphBuilder.RegisterExternalTexture(AlbedoAtlas);
   FrameTemporaries.NormalAtlas   = GraphBuilder.RegisterExternalTexture(NormalAtlas);
   // ... 他のアトラス
   ```

---

## FreeSurfaceCacheAllocation

```cpp
void FreeSurfaceCacheAllocation(
    FLumenCard& Card,
    FLumenSurfaceCacheAllocator& Allocator);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `Card` | `FLumenCard&` | 解放対象の Card |
| `Allocator` | `FLumenSurfaceCacheAllocator&` | 物理ページアロケータ |

### 使用箇所
- [[ref_lumen_scene]] `UpdateSurfaceCacheAllocationState()` — デバッグリセット時
- [[ref_lumen_scene_data]] `FLumenSceneData::RemoveCardFromAtlas()` — Card 削除時

### 内部動作
```cpp
void FreeSurfaceCacheAllocation(FLumenCard& Card, FLumenSurfaceCacheAllocator& Allocator)
{
    for (int32 ResLevel = Card.MinAllocatedResLevel; ResLevel <= Card.MaxAllocatedResLevel; ResLevel++) {
        FLumenSurfaceMipMap& MipMap = Card.GetMipMap(ResLevel);
        if (MipMap.IsAllocated()) {
            // ページテーブルの各エントリを解放
            for (int32 i = 0; i < MipMap.PageTableSpanSize; i++) {
                FLumenPageTableEntry& PageEntry = PageTable[MipMap.PageTableSpanOffset + i];
                if (PageEntry.IsMapped()) {
                    Allocator.Free(PageEntry);
                    PageEntry.PhysicalPageCoord = FIntPoint(-1, -1);
                }
            }
            // ページテーブルスパンの解放
            PageTable.RemoveSpan(MipMap.PageTableSpanOffset, MipMap.PageTableSpanSize);
            MipMap.PageTableSpanOffset = -1;
            MipMap.PageTableSpanSize   = 0;
        }
    }
    Card.UpdateMinMaxAllocatedLevel();
}
```

---

<details>
<summary>GetSurfaceCacheCompression — 圧縮方式の取得</summary>

```cpp
ESurfaceCacheCompression GetSurfaceCacheCompression();
```

### 戻り値
`ESurfaceCacheCompression` — 現在のプラットフォームとドライバで使用できる圧縮方式

### 内部動作
```cpp
ESurfaceCacheCompression GetSurfaceCacheCompression()
{
    if (!CVarLumenSurfaceCacheCompress.GetValueOnRenderThread()) {
        return ESurfaceCacheCompression::Disabled;
    }
    // DX12 + UAV エイリアシングサポートなら UAVAliasing
    if (GRHISupportsUAVAliasing && GRHISupportsBC7Compression) {
        return ESurfaceCacheCompression::UAVAliasing;
    }
    // それ以外は CopyTextureRegion フォールバック
    return ESurfaceCacheCompression::CopyTextureRegion;
}
```

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::AllocateCardAtlases()` — アトラスフォーマット選択
- [[ref_lumen_scene]] `UpdateLumenSurfaceCacheAtlas()` — コピー方式の選択

</details>

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.LumenScene.SurfaceCache.AtlasSize` | 4096 | アトラスの一辺のサイズ（px）|
| `r.LumenScene.SurfaceCache.CardCapturesPerFrame` | 300 | 1 フレームあたり最大キャプチャ Card 数 |
| `r.LumenScene.SurfaceCache.CardCaptureFactor` | 64 | キャプチャテクセル上限 = 総テクセル数 / Factor |
| `r.LumenScene.SurfaceCache.CardCaptureRefreshFraction` | 0.125 | 既存ページ再キャプチャに使えるバジェット割合 |
| `r.LumenScene.SurfaceCache.RemovesPerFrame` | 512 | 1 フレームあたり最大 Card 削除数 |
| `r.LumenScene.SurfaceCache.Freeze` | 0 | Surface Cache 更新を凍結（デバッグ用）|
| `r.LumenScene.SurfaceCache.Reset` | 0 | Surface Cache を全リセット（デバッグ用）|
| `r.LumenScene.SurfaceCache.ResetEveryNthFrame` | 0 | N フレームごとに全リセット（デバッグ用）|
| `r.LumenScene.SurfaceCache.CardFixedDebugResolution` | -1 | 全 Card を固定解像度にする（-1 = 無効）|
| `r.LumenScene.SurfaceCache.Compress` | 1 | BC 圧縮を使うか（1 = 使用、0 = 無効）|
