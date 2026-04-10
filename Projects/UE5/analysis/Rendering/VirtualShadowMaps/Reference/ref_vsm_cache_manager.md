# REF: VirtualShadowMapCacheManager.h / .cpp

- 対象ファイル: `Private/VirtualShadowMaps/VirtualShadowMapCacheManager.h` / `.cpp`
- 関連Details: [[b_vsm_cache_manager]]

---

## クラス・構造体

### `FVirtualShadowMapCacheKey`

キャッシュエントリのキー。

```cpp
struct FVirtualShadowMapCacheKey
{
    uint32 ViewUniqueID;   // ビューの一意ID（ローカルライトのキューブ面等）
    int32  LightSceneId;   // ライトのシーンID
    uint32 ShadowTypeId;   // シャドウ種別（クリップマップ vs ローカル等）
};
```

---

### `FVirtualShadowMapCacheEntry`

1つのVSM（またはクリップマップレベル）のキャッシュデータ。

| フィールド | 型 | 説明 |
|-----------|---|------|
| `ProjectionData` | `FVirtualShadowMapProjectionShaderData` | 現フレームの投影データ |
| `PrevHZBMetadata` | `FVirtualShadowMapHZBMetadata` | 前フレームのHZBメタデータ |
| `CurrentHZBMetadata` | `FVirtualShadowMapHZBMetadata` | 現フレームのHZBメタデータ |
| `NextData` | `FVirtualShadowMapCacheEntry::FNextFrameData` | 次フレーム用データ |
| `Clipmap` | `FClipmapInfo` | クリップマップ固有情報（レベル・CornerOffset等） |

**メソッド:**

| メソッド | 説明 |
|---------|------|
| `Update(int32 VirtualShadowMapId)` | VSM IDを更新し現フレーム用に初期化 |
| `UpdateClipmapLevel(int32 Level, FVector CornerOffset, ...)` | クリップマップレベル情報更新 |
| `SetHZBViewParams(const FViewMatrices&)` | HZBビューパラメータ設定 |
| `UpdateHZBMetadata(FViewMatrices, FIntRect, int32 LayerIndex)` | HZBメタデータ更新 |

---

### `FVirtualShadowMapHZBMetadata`

HZB用のビュー情報（カメラが動いた際に無効化判定に使用）。

```cpp
struct FVirtualShadowMapHZBMetadata
{
    FViewMatrices ViewMatrices;   // ビュー行列
    FIntRect      ViewRect;       // ビューポート矩形
    int32         TargetLayerIndex; // 物理プールのレイヤーインデックス
    bool          bMatricesDirty; // 行列が変更されたか
};
```

---

### `FVirtualShadowMapPerLightCacheEntry`

1ライト分のキャッシュエントリ。複数の `FVirtualShadowMapCacheEntry` を保持。

| フィールド | 型 | 説明 |
|-----------|---|------|
| `ShadowMapEntries` | `TArray<FVirtualShadowMapCacheEntry>` | 各レベル/面のキャッシュエントリ |
| `RenderedPrimitives` | `TBitArray<>` | 描画済みプリミティブのビットマスク |
| `RenderedFrameNumber` | `int32` | 最後に描画されたフレーム番号 |
| `ScheduledFrameNumber` | `int32` | 描画予定のフレーム番号 |
| `VirtualShadowMapId` | `int32` | 現在のVSM ID |
| `PrevVirtualShadowMapId` | `int32` | 前フレームのVSM ID |
| `bIsUncached` | `bool` | キャッシュ無効（強制再描画） |
| `bIsDistantLight` | `bool` | 遠方ライト扱い |
| `bUseReceiverMask` | `bool` | レシーバーマスク使用 |
| `bReferencedThisRender` | `bool` | このフレームで参照されたか |

**メソッド:**

| メソッド | 説明 |
|---------|------|
| `Update(FVirtualShadowMapArray&, int32 NumShadowMaps)` | 基本更新（ローカルライト用） |
| `UpdateClipmap(FVirtualShadowMapArray&, FVirtualShadowMapClipmap&)` | クリップマップ更新 |
| `UpdateLocal(FVirtualShadowMapArray&, FProjectedShadowInfo&, ...)` | ローカルライト更新 |
| `OnPrimitiveRendered(const FPrimitiveSceneInfo*)` | プリミティブ描画記録 |
| `IsFullyCached() const` | 完全キャッシュ済みか |
| `IsUncached() const` | キャッシュなし（強制再描画）か |
| `MarkRendered(int32 FrameNumber)` | 描画済みとしてマーク |
| `Invalidate()` | キャッシュ無効化 |
| `MarkScheduled()` | 描画スケジュール済みマーク |
| `UpdateVirtualShadowMapId(int32 Id)` | VSM ID更新 |
| `AffectsBounds(const FBoxSphereBounds&) const` | バウンド判定（無効化チェック） |

---

### `FVirtualShadowMapArrayFrameData`

1フレーム分のGPUリソース（CacheManagerが保持し、フレーム間で引き渡す）。

| フィールド | 型 | 説明 |
|-----------|---|------|
| `PageTable` | `TRefCountPtr<IPooledRenderTarget>` | 仮想→物理マッピングテクスチャ |
| `PageFlags` | `TRefCountPtr<FRDGPooledBuffer>` | ページ状態フラグ |
| `UncachedPageRectBounds` | `TRefCountPtr<FRDGPooledBuffer>` | 非キャッシュページAABB |
| `AllocatedPageRectBounds` | `TRefCountPtr<FRDGPooledBuffer>` | 割り当てページAABB |
| `ProjectionData` | `TRefCountPtr<FRDGPooledBuffer>` | 投影データ配列 |
| `PhysicalPageLists` | `TRefCountPtr<FRDGPooledBuffer>` | 物理ページリスト |
| `PageRequestFlags` | `TRefCountPtr<FRDGPooledBuffer>` | ページ要求フラグ |
| `NanitePerformanceFeedback` | `TRefCountPtr<FRDGPooledBuffer>` | Nanite性能フィードバック |

**メソッド:**

| メソッド | 説明 |
|---------|------|
| `GetGPUSizeBytes() const` | フレームデータのGPUメモリ使用量 |

---

### `FPhysicalPageMetaData`

物理ページ1枚分のメタデータ（CPUサイド）。

```cpp
struct FPhysicalPageMetaData
{
    uint32 Flags;                          // ページ状態フラグ
    uint32 LastRequestedSceneFrameNumber;  // 最後に要求されたフレーム
    int32  VirtualShadowMapId;             // 所属VSM ID
    uint8  MipLevel;                       // MIPレベル
    FIntPoint PageAddress;                 // 物理アドレス（2D座標）
};
```

---

### `FVirtualShadowMapArrayCacheManager`

**`ISceneExtension` 継承**。物理ページプールを所有し永続管理するメインクラス。

**主要メソッド:**

| メソッド | 説明 |
|---------|------|
| `FindCreateLightCacheEntry(int32 LightSceneId, uint32 ViewUniqueID, uint32 NumShadowMaps, uint32 TypeId)` | キャッシュエントリ取得または新規作成 |
| `SetPhysicalPoolSize(FRDGBuilder&, FIntPoint, int ArraySize, uint32 MaxPhysicalPages)` | 物理プールサイズ設定（リサイズ時はキャッシュ無効化） |
| `SetHZBPhysicalPoolSize(FRDGBuilder&, FIntPoint, EPixelFormat)` | HZB物理プールサイズ設定 |
| `UpdateUnreferencedCacheEntries(FVirtualShadowMapArray&)` | オフスクリーンライトのキャッシュ更新 |
| `ExtractFrameData(FRDGBuilder&, FVirtualShadowMapArray&, const FSceneRenderer&, bool bAllowPersistent)` | フレーム終了時にRDGリソースを永続バッファへ |
| `ProcessInvalidations(FRDGBuilder&, FSceneUniformBuffer&, FInvalidatingPrimitiveCollector&)` | GPU上でキャッシュページを無効化 |
| `Invalidate()` | 全キャッシュ強制無効化 |
| `IsCacheEnabled() const` | キャッシュ機能が有効か |
| `IsCacheDataAvailable() const` | 前フレームデータが利用可能か |
| `IsHZBDataAvailable() const` | HZBデータが利用可能か |
| `GetPhysicalPagePool() const` | 物理ページプールのリソース取得 |
| `GetPhysicalPageMetaData()` | ページメタデータ配列取得 |
| `GetGPUSizeBytes(bool bLogSizes) const` | 総GPUメモリ使用量取得 |

---

### `FVirtualShadowMapInvalidationSceneUpdater`

**`ISceneExtensionUpdater` 継承**。シーン変更イベントをフックして無効化を収集。

| メソッド | 説明 |
|---------|------|
| `PreLightsUpdate(FRDGBuilder&, const FScenePreUpdateChangeSet&)` | ライト更新前処理 |
| `PreSceneUpdate(FRDGBuilder&, const FScenePreUpdateChangeSet&)` | シーン更新前処理 |
| `PostSceneUpdate(FRDGBuilder&, const FScenePostUpdateChangeSet&)` | シーン更新後処理（移動等を収集） |
| `PostGPUSceneUpdate(FRDGBuilder&, FSceneUniformBuffer&)` | GPUScene更新後に無効化を実行 |

---

### `FInvalidatingPrimitiveCollector`

無効化対象プリミティブを収集するヘルパー。

| メソッド | 説明 |
|---------|------|
| `Added(const FPrimitiveSceneInfo*)` | 新規プリミティブ追加通知 |
| `Removed(const FPrimitiveSceneInfo*)` | プリミティブ削除通知 |
| `UpdatedTransform(const FPrimitiveSceneInfo*)` | トランスフォーム変更通知 |
| `AddPrimitivesToInvalidate(...)` | 無効化対象を一括追加 |

---

### `FVirtualShadowMapFeedback`

GPU→CPUフィードバックバッファ管理（Nanite性能統計等）。

| メソッド | 説明 |
|---------|------|
| `SubmitFeedbackBuffer(FRDGBuilder&, FRDGBufferRef)` | GPUフィードバックバッファ送信 |
| `GetLatestReadbackBuffer()` | 最新のCPUリードバックバッファ取得 |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Shadow.Virtual.Cache` | 1 | キャッシュ有効/無効 |
| `r.Shadow.Virtual.Cache.StaticSeparate` | 1 | 静的ページを分離保持 |
| `r.Shadow.Virtual.Cache.InvalidateUseHZB` | 1 | HZBを使った無効化精度向上 |
| `r.Shadow.Virtual.Cache.DeformableMeshesInvalidate` | 1 | 変形メッシュの無効化 |
| `r.Shadow.Virtual.Cache.MaxPageAgeSinceLastRequest` | 1000 | ページ保持最大フレーム数 |
| `r.Shadow.Virtual.Cache.MaxLightAgeSinceLastRequest` | 10 | ライトキャッシュ保持フレーム数 |
| `r.Shadow.Virtual.Cache.FramesStaticThreshold` | 100 | 静的キャッシュ昇格フレーム数 |
| `r.Shadow.Virtual.Cache.AllocateViaLRU` | 1 | LRUベースページ割り当て |
| `r.Shadow.Virtual.Cache.CPUCullInvalidationsOutsideLightRadius` | 1 | ライト半径外のCPUカリング |
| `r.Shadow.Virtual.AllocatePagePoolAsReservedResource` | 1 | 予約リソースとしてプール確保 |
| `r.Shadow.Virtual.DynamicRes.MaxResolutionLodBias` | 2.0 | 動的解像度最大LODバイアス |
| `r.Shadow.Virtual.DynamicRes.MaxComputeResolutionLodBiasLocal` | 99999.0 | ローカル最大コンピュートLODバイアス |
| `r.Shadow.Virtual.DynamicRes.MaxComputeResolutionLodBiasDirectional` | 99999.0 | ディレクショナル最大コンピュートLODバイアス |
