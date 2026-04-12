# REF: VSM Cache Invalidation シェーダー

- グループ: a - Invalidation
- 詳細: [[detail_invalidation]]
- CPU リファレンス: [[ref_vsm_cache_manager]]
- ソース: `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapCacheGPUInvalidation.usf`
          `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapCacheLoadBalancer.usf`

---

## VirtualShadowMapCacheGPUInvalidation.usf

### VSMUpdateViewInstanceStateCS

```hlsl
[numthreads(THREAD_GROUP_SIZE, 1, 1)]
void VSMUpdateViewInstanceStateCS(
    uint3 GroupId           : SV_GroupID,
    uint GroupThreadIndex   : SV_GroupIndex,
    uint3 DispatchThreadId  : SV_DispatchThreadID
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 各インスタンスの動的（Deforming）/静的状態を 32bit ワード単位でビット演算で差分検出し、状態変化インスタンスを無効化キューに追加 |
| **入力** | `InOutViewInstanceState`（永続状態バッファ）, `SceneRendererViewId`, `MaxValidInstanceIndex` |
| **出力** | `InOutViewInstanceState`（更新済み）, `OutCacheInstanceAsDynamic`（トランジェント）, `OutInvalidationQueue`, `OutInvalidationArgs` |
| **CPU 関数** | `FVirtualShadowMapCacheManager::ProcessInvalidations()` |
| **特徴** | 1スレッドで 32 インスタンスを処理。`FOREACH_SET_BIT_BEGIN/END` マクロで変化ビットのみ処理 |

### ProcessInvalidationQueueGPUCS

```hlsl
[numthreads(THREAD_GROUP_SIZE, 1, 1)]
void ProcessInvalidationQueueGPUCS(
    uint3 GroupId           : SV_GroupID,
    uint GroupThreadIndex   : SV_GroupIndex,
    uint3 DispatchThreadId  : SV_DispatchThreadID
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 無効化キューの各インスタンスについて、全 VSM をスキャンして該当ページを動的無効化 |
| **入力** | `InvalidationQueue`（インスタンスIDリスト）, `InvalidationArgs`（件数）, VSM ProjectionData |
| **出力** | 各 VSM の動的ページフラグを無効化（`InvalidateInstancePages()` 内で書き込み）|
| **CPU 関数** | `FVirtualShadowMapCacheManager::ProcessInvalidations()`（Indirect Dispatch）|
| **注意** | 全 VSM ループはブルートフォース。無効化インスタンスが少ない（通常ケース）なら問題なし |

### VSMResetInstanceStateCS

```hlsl
[numthreads(NUM_THREADS_PER_GROUP, 1, 1)]
void VSMResetInstanceStateCS(
    uint3 GroupId           : SV_GroupID,
    uint GroupThreadIndex   : SV_GroupIndex
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 追加・削除されたインスタンスの状態フラグを `InOutViewInstanceState` からリセット |
| **入力** | `InstanceCullingLoadBalancer`（追加/削除インスタンスリスト）|
| **出力** | `InOutViewInstanceState`（フラグクリア）|
| **CPU 関数** | `FVirtualShadowMapCacheManager::ProcessInvalidations()` |

---

## VirtualShadowMapCacheLoadBalancer.usf

### InvalidateInstancePagesLoadBalancerCS

```hlsl
[numthreads(CS_1D_GROUP_SIZE_X, 1, 1)]
void InvalidateInstancePagesLoadBalancerCS(
    uint3 GroupId           : SV_GroupID,
    uint GroupThreadIndex   : SV_GroupIndex,
    uint3 DispatchThreadId  : SV_DispatchThreadID
)
```

| 項目 | 内容 |
|-----|------|
| **目的** | `InstanceCullingLoadBalancer` を使って無効化対象インスタンスを負荷分散しながら各 VSM ページに対してテスト・無効化 |
| **入力** | `InstanceCullingLoadBalancer`（インスタンスバッチ情報）, `FInstanceInvalidationPayload`（無効化フラグ）|
| **出力** | 対応する VSM ページを無効化（`InvalidateInstancePages()` 呼び出し）|
| **CPU 関数** | `FVirtualShadowMapCacheManager::ProcessInvalidations()` |
| **使用場面** | 大量のインスタンスが同時に追加/移動された場合の高スループット無効化 |

---

## 共有ヘッダー

| ファイル | 役割 |
|---------|------|
| `VirtualShadowMapCacheInvalidation.ush` | `InvalidateInstancePages()` / `PackInvalidationItem()` / `UnpackInvalidationItem()` インライン実装 |
| `VirtualShadowMapPageCacheCommon.ush` | `VSM_EXTENDED_FLAG_*` フラグ定数 / `FVirtualSMCachedPage` 構造体 |
| `VirtualShadowMapHandle.ush` | `FVirtualShadowMapHandle` 型（ID エンコーディング）|

---

> [!note]- 32bit ワード単位のビット演算による効率化
> `VSMUpdateViewInstanceStateCS` は 1 スレッドに 32 インスタンスを割り当て、ビットマスク演算（XOR, AND）で  
> 状態変化を一括検出する。通常フレームでは状態変化インスタンスはほぼゼロなので、  
> `FOREACH_SET_BIT_BEGIN/END`（= 変化ビットを走査するループ）がほとんど実行されない。  
> これにより何千インスタンスのシーンでもカーネルがほぼ O(N/32) で完走できる。

> [!note]- 静的ページを保持する理由（Dynamic のみ無効化）
> VSM には「静的キャッシュページ」（毎フレーム再描画しない）と「動的ページ」（毎フレーム更新）がある。  
> インスタンスが Static → Dynamic に遷移した場合、そのインスタンスは動的ページに再描画される。  
> 静的ページには古いシャドウデータが残るが、動的ページが上から合成されるため視覚的に問題ない。  
> 逆に Dynamic → Static の遷移では無効化不要（次フレームで自然に静的ページに移行する）。

> [!note]- ProcessInvalidationQueueGPUCS のブルートフォースが許容される理由
> 無効化キューに入るインスタンスは「変形状態が変化したもの」のみ（通常は少数）。  
> そのため全 VSM × キューアイテム数のスキャンは実際には軽い。  
> インスタンス数が多くても、状態変化インスタンスは少ないという設計前提が成立する限り、  
> LoadBalancer（`InvalidateInstancePagesLoadBalancerCS`）が高コストケースをカバーする。
