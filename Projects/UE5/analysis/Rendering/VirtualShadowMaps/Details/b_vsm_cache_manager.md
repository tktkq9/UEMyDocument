# VSM b: VirtualShadowMapCacheManager（キャッシュ・無効化）

- 対象: `VirtualShadowMaps/VirtualShadowMapCacheManager.h/.cpp`
- 上位: [[04_vsm_overview]]

---

## 役割

`FVirtualShadowMapArrayCacheManager` は **フレームをまたいで VSM データを永続管理する**クラス。  
`ISceneExtension` を継承し、シーンの一部として登録される。

- **物理ページプール（PhysicalPagePool）を所有**し、フレームごとの `FVirtualShadowMapArray` に貸し出す
- ライトごとのキャッシュエントリ（`FVirtualShadowMapPerLightCacheEntry`）を保持
- シーン更新（プリミティブ追加・移動・削除）をフックして**必要なページのみ無効化**

---

## クラス階層

```
ISceneExtension (プラグインポイント)
└── FVirtualShadowMapArrayCacheManager
        ├── 物理ページプール（Texture2DArray, 永続）
        ├── HZB物理プール
        ├── FVirtualShadowMapArrayFrameData（前フレームデータ）
        └── TMap<FVirtualShadowMapCacheKey, FVirtualShadowMapPerLightCacheEntry>

ISceneExtensionUpdater
└── FVirtualShadowMapInvalidationSceneUpdater
        ├── PreLightsUpdate()    … ライト更新前処理
        ├── PreSceneUpdate()     … シーン更新前処理
        ├── PostSceneUpdate()    … シーン更新後処理
        └── PostGPUSceneUpdate() … GPUScene更新後の無効化実行
```

---

## キャッシュエントリ構造

```
FVirtualShadowMapCacheKey
  { ViewUniqueID, LightSceneId, ShadowTypeId }
       ↓ map key
FVirtualShadowMapPerLightCacheEntry   ← ライト1つに対応
  ├── ShadowMapEntries[]              ← クリップマップ各レベル or ローカル
  │     └── FVirtualShadowMapCacheEntry
  │           ├── ProjectionData（投影行列等）
  │           ├── PrevHZBMetadata / CurrentHZBMetadata
  │           └── NextData（次フレーム用）
  ├── RenderedPrimitives (TBitArray)  ← 描画済みプリミティブ記録
  ├── RenderedFrameNumber             ← 最後に描画されたフレーム
  ├── ScheduledFrameNumber
  ├── bIsUncached                     ← 強制再描画フラグ
  ├── bIsDistantLight                 ← 遠方ライト（省メモリ扱い）
  └── bUseReceiverMask
```

---

## 静的キャッシュの仕組み

```
ページは2種類:

[Static Page]
  - 静的なジオメトリのみが投影するページ
  - r.Shadow.Virtual.Cache.StaticSeparate=1 で動的と分離保持
  - 静的オブジェクトが動かない限り再描画しない
  - 必要ならLRUで解放

[Dynamic Page]
  - 動的オブジェクト（スケルタルメッシュ等）が投影するページ
  - 毎フレーム再描画が必要な場合あり
```

---

## 無効化パイプライン

```
FInvalidatingPrimitiveCollector::Added()     … 新規プリミティブ → 無効化
FInvalidatingPrimitiveCollector::Removed()   … 削除プリミティブ → 無効化
FInvalidatingPrimitiveCollector::UpdatedTransform() … 移動 → バウンドチェックして無効化

  ↓ ProcessInvalidations() で GPU に送信

GPU 側でページフラグをクリア
  → 次フレームで BuildPageAllocations() が再描画扱いにする
```

---

## フレーム間データ転送

```
【描画フレーム終了時】
FVirtualShadowMapArray::PostRender()
  → ExtractFrameData() でフレームデータをCacheManagerに保存
     ・PageTable / PageFlags / ProjectionData 等を永続バッファにコピー

【次フレーム開始時】
FVirtualShadowMapArray::Initialize()
  → CacheManager から前フレームデータを受け取る
  → キャッシュ済みページは再描画をスキップ
```

---

## LRU ページ割り当て

物理ページプールが満杯になった場合、  
`r.Shadow.Virtual.Cache.AllocateViaLRU=1` で最も古く使われていないページを解放して再利用する。

```
FPhysicalPageMetaData (ページごとのメタデータ)
  ├── LastRequestedSceneFrameNumber … 最後に要求されたフレーム
  ├── VirtualShadowMapId            … 所属VSM
  ├── MipLevel
  └── Flags
```

---

## コード実行フロー

### エントリポイント（無効化パイプライン）

```
FScene::UpdateAllPrimitives()
  └─ FVirtualShadowMapInvalidationSceneUpdater::PostSceneUpdate()
       └─ FInvalidatingPrimitiveCollector::UpdatedTransform() / Added() / Removed()

FVirtualShadowMapInvalidationSceneUpdater::PostGPUSceneUpdate()
  └─ FVirtualShadowMapArrayCacheManager::ProcessInvalidations()  VirtualShadowMapCacheManager.cpp:1883
       ├─ FInstancePageInvalidationCS (GPU でページフラグをクリア)
       └─ キャッシュ済みページを無効化 → 次フレームで再描画扱いになる
```

### エントリポイント（フレーム間データ転送）

```
FVirtualShadowMapArray::PostRender()                VirtualShadowMapArray.cpp:1729
  └─ FVirtualShadowMapArrayCacheManager::ExtractFrameData()  VirtualShadowMapCacheManager.cpp:1456
       ├─ RDG バッファを PooledBuffer に抽出（永続化）
       ├─ PhysicalPageMetaData を更新（LRU 情報）
       └─ PerLightCacheEntry::UpdateCachedFrameData() → 次フレーム用データ保存

次フレーム Initialize()
  └─ CacheManager から前フレームデータを受け取る
       └─ キャッシュ済みページは BuildPageAllocations() でスキップ
```

### フロー詳細

1. **PostSceneUpdate()** — プリミティブのトランスフォーム変化・追加・削除を `FInvalidatingPrimitiveCollector` に収集する
2. **ProcessInvalidations()** (`VirtualShadowMapCacheManager.cpp:1883`) — 収集した変化プリミティブのバウンドと物理ページの投影範囲を GPU 上で比較し、重なりがあるページフラグをクリアする
3. **ExtractFrameData()** (`VirtualShadowMapCacheManager.cpp:1456`) — フレーム終了時に RDG バッファを `FVirtualShadowMapArrayFrameData` に抽出して永続化する
4. 次フレーム `Initialize()` で前フレームデータを受け取り、キャッシュ済みページをスキップして差分のみ再描画する

### 関与クラス・関数一覧

| クラス/関数 | ファイル:行 | 役割 |
|------------|-----------|------|
| `FVirtualShadowMapInvalidationSceneUpdater::PostGPUSceneUpdate()` | `VirtualShadowMapCacheManager.cpp` | GPUScene 更新後に無効化実行 |
| `FVirtualShadowMapArrayCacheManager::ProcessInvalidations()` | `VirtualShadowMapCacheManager.cpp:1883` | GPU でページフラグをクリア |
| `FVirtualShadowMapArrayCacheManager::ExtractFrameData()` | `VirtualShadowMapCacheManager.cpp:1456` | フレームデータ永続化 |
| `FVirtualShadowMapArrayCacheManager::SetPhysicalPoolSize()` | `VirtualShadowMapCacheManager.cpp:1137` | 物理プールリサイズ（リサイズ時は全キャッシュ無効化）|
| `FInvalidatingPrimitiveCollector` | `VirtualShadowMapCacheManager.h` | 無効化対象プリミティブの収集 |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Shadow.Virtual.Cache` | 1 | キャッシュ機能全体の有効/無効 |
| `r.Shadow.Virtual.Cache.StaticSeparate` | 1 | 静的ページを動的と分離保持 |
| `r.Shadow.Virtual.Cache.MaxPageAgeSinceLastRequest` | 1000 | ページを保持する最大フレーム数 |
| `r.Shadow.Virtual.Cache.MaxLightAgeSinceLastRequest` | 10 | ライトキャッシュを保持するフレーム数 |
| `r.Shadow.Virtual.Cache.FramesStaticThreshold` | 100 | 静的キャッシュに昇格するまでのフレーム数 |
| `r.Shadow.Virtual.Cache.AllocateViaLRU` | 1 | LRUベース物理ページ割り当て |
| `r.Shadow.Virtual.Cache.InvalidateUseHZB` | 1 | HZBを使った無効化精度向上 |
| `r.Shadow.Virtual.Cache.DeformableMeshesInvalidate` | 1 | 変形メッシュの無効化 |
| `r.Shadow.Virtual.Cache.CPUCullInvalidationsOutsideLightRadius` | 1 | ライト半径外の無効化をCPUでカリング |
| `r.Shadow.Virtual.AllocatePagePoolAsReservedResource` | 1 | 予約リソースとしてプールを確保 |

---

## 関連リファレンス

- [[ref_vsm_cache_manager]] — FVirtualShadowMapArrayCacheManager / FVirtualShadowMapPerLightCacheEntry リファレンス
- [[ref_vsm_array]] — FVirtualShadowMapArray リファレンス
