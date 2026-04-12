# VSM Cache Invalidation シェーダー詳細

- グループ: a - Invalidation
- GPU 概要: [[01_vsm_gpu_overview]]
- CPU 詳細: [[b_vsm_cache_manager]]
- リファレンス: [[ref_invalidation]]

---

## 概要

VSM のキャッシュ無効化パイプライン。  
インスタンスが「変形（Deforming）」状態に変化したかを GPU 上で検出し、  
影響する VSM ページを選択的に無効化して再レンダリングをトリガーする。  
CPU 側で全インスタンスを調べるコストを避けるため、GPU 上で差分検出を行う設計。

---

## パス構成

```
[1] VSMUpdateViewInstanceStateCS（VirtualShadowMapCacheGPUInvalidation.usf）
     → インスタンスごとの動的/静的状態を 32 ビットワード単位で差分検出
     → 状態変化インスタンスを OutInvalidationQueue に Append

[2] ProcessInvalidationQueueGPUCS（VirtualShadowMapCacheGPUInvalidation.usf）
     → InvalidationQueue の各インスタンスに対して全 VSM をブルートフォース検索
     → 該当 VSM の動的ページを InvalidateInstancePages() で無効化

[3] InvalidateInstancePagesLoadBalancerCS（VirtualShadowMapCacheLoadBalancer.usf）
     → インスタンスの Bounds を各 VSM に対してテストして負荷分散で無効化

[4] VSMResetInstanceStateCS（VirtualShadowMapCacheGPUInvalidation.usf）
     → 追加/削除されたインスタンスの状態フラグをリセット
```

---

## VSMUpdateViewInstanceStateCS のコアロジック

32 インスタンスを 1 スレッドで処理（1 uint ワード = 32 インスタンスのビットマスク）。

```hlsl
[numthreads(THREAD_GROUP_SIZE, 1, 1)]
void VSMUpdateViewInstanceStateCS(uint3 GroupId, uint GroupThreadIndex, uint3 DispatchThreadId)
{
    uint WordIndex = DispatchThreadId.x;

    // ビットマスク操作で 32 インスタンスを一括処理
    uint IsDeformingMask     = LoadInstanceDeformingWord(WordIndex, SceneRendererViewId);
    uint CachedAsDynamicMask = InOutViewInstanceState[WordIndex];   // 前フレームの状態
    uint TrackedMask         = InOutViewInstanceState[WordIndex + StateWordStride];

    // 状態変化 = (新状態 XOR 旧状態) AND 追跡済み
    uint StateChangeMask = (IsDeformingMask ^ CachedAsDynamicMask) & TrackedMask;

    // Dynamic に変化したインスタンスのみ無効化キューへ
    uint InvalidationMask = StateChangeMask & IsDeformingMask;

    FOREACH_SET_BIT_BEGIN(InvalidationMask, BitIndex, BitMask)
    {
        // WaveInterlockedAddScalarInGroups でカウンタを安全インクリメント
        WaveInterlockedAddScalarInGroups(OutInvalidationArgs[3], OutInvalidationArgs[0],
                                          THREAD_GROUP_SIZE, 1, WriteOffset);
        OutInvalidationQueue[WriteOffset] = PackInvalidationItem(
            WordIndex * 32u + BitIndex, SceneRendererViewId, ...);
    }
    FOREACH_SET_BIT_END()

    // 永続バッファを現フレームの状態に更新
    InOutViewInstanceState[WordIndex] = IsDeformingMask;
    InOutViewInstanceState[WordIndex + StateWordStride] = AllInstancesMask;
}
```

**Static → Dynamic 遷移のみ無効化する理由：**  
Dynamic → Static の遷移は静的ページが Dynamic よりも優先的にサンプリングされるため、  
即座の無効化は不要（次フレーム以降に自然にキャッシュが置き換わる）。

---

## ProcessInvalidationQueueGPUCS のコアロジック

```hlsl
[numthreads(THREAD_GROUP_SIZE, 1, 1)]
void ProcessInvalidationQueueGPUCS(...)
{
    FInvalidationItem Item = UnpackInvalidationItem(InvalidationQueue[InvalidationQueueIndex]);
    FInstanceSceneData InstanceSceneData = GetInstanceSceneData(Item.InstanceId);

    // 全 VSM をブルートフォースで検索（通常はライト数 = 数十〜百程度）
    for (uint VirtualShadowMapIndex = 0; VirtualShadowMapIndex < VirtualShadowMap.NumFullShadowMaps; ++VirtualShadowMapIndex)
    {
        FVirtualShadowMapProjectionShaderData ProjectionData = GetVirtualShadowMapProjectionData(Handle);

        // ビューIDが一致する VSM のみを処理
        if (SceneRendererPrimaryViewId != Item.SceneRendererPrimaryViewId)
            continue;

        // 動的ページのみを無効化（静的ページは保持）
        uint OverrideFlags = VSM_EXTENDED_FLAG_INVALIDATE_DYNAMIC;
        InvalidateInstancePages(ProjectionData, InstanceSceneData, OverrideFlags);
    }
}
```

---

## InvalidateInstancePagesLoadBalancerCS のコアロジック

`InvalidateInstancePagesLoadBalancerCS` は `CullPerPageDrawCommandsCs` と同様のインスタンスカリングロードバランサーを使用して、  
インスタンスの Bounds を VSM の各ページに対してテストする負荷分散版。  
インスタンス数が多い場合（大量の動的オブジェクトが追加される等）に効率的。

```hlsl
[numthreads(CS_1D_GROUP_SIZE_X, 1, 1)]
void InvalidateInstancePagesLoadBalancerCS(...)
{
    // InstanceCullingLoadBalancer で InstanceId を取得
    // インスタンスの AABB と VSM ページ境界を判定
    // 重なるページを OutInvalidatedPages に書き込み
}
```

---

## 入出力

| リソース | 内容 |
|---------|------|
| `InOutViewInstanceState` | インスタンスの動的/静的フラグ（32bit ワード配列、永続）|
| `OutInvalidationQueue` | 無効化対象インスタンスの ID リスト（一時バッファ）|
| `OutInvalidationArgs` | Indirect Dispatch 引数（`ProcessInvalidationQueueGPUCS` 用）|
| `OutCacheInstanceAsDynamic` | シーンレンダラー向け現フレーム状態（トランジェント）|
| `VirtualShadowMap.NumFullShadowMaps` | 処理対象 VSM 数（全ライト分）|

---

## CPU 呼び出しの流れ

```
FVirtualShadowMapCacheManager::ProcessInvalidations()   // VirtualShadowMapCacheManager.cpp
  │
  ├─ VSMUpdateViewInstanceStateCS
  │    → 状態変化インスタンスを OutInvalidationQueue に Append
  │
  ├─ ProcessInvalidationQueueGPUCS（Indirect Dispatch）
  │    → 無効化キューの各インスタンスについて全 VSM をスキャン
  │
  └─ InvalidateInstancePagesLoadBalancerCS
       → Bounds ベースで負荷分散しながらページを無効化
```
