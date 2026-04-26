---
name: VSM Page Marking & Streaming
description: VSM のフレーム間ページキャッシュ・無効化・ストリーミング戦略
type: project
---

# VSM Page Marking & Streaming（フレーム間ページキャッシュ）

- 上位: [[_algorithm_index]]
- 関連: [[vsm_overview]] / [[nanite_cluster_culling]]
- 採用システム: VSM 全体（特に静的 / 半静的シーン）
- 出典 ID: **S21**（[[_source_index]]）— §4 Caching
- 影響元: Sparse Virtual Texturing 系のキャッシュ戦略

---

## 1. 何のためのアルゴリズムか

VSM の物理ページ（典型 2048 個）を毎フレーム全部再ラスタすると Nanite でも重い。実際は:
- **シーンの大部分は動かない**（建物、地形、家具）
- 動くのはキャラ・乗り物・葉揺れなど局所
- ライトも多くは静的 / 半静的

→ **動かないページはキャッシュして再利用**、変化したものだけ再ラスタ。

### 素朴な手法の問題

- **毎フレーム全ページ再ラスタ**: 静的シーンでも常時フルコスト
- **完全キャッシュ**: 動的物体に追従できない
- **CPU dirty tracking**: 数千インスタンスでスケールせず

### Lumen / Nanite 流の貢献

- ページ単位 dirty フラグ + GPU 駆動 invalidation
- **"Static Caching"**: 一定フレーム動かないオブジェクトは静的バケットに昇格、独立ページに分離
- HZB ベース invalidation: 動いたインスタンスでも HZB 上で見えなければスキップ

---

## 2. 理論

### 2.1 ページ状態

各物理ページに以下の状態を持つ:

| 状態 | 意味 |
|------|------|
| Free | 未使用、フリーリストに在籍 |
| Allocated (Cached) | 過去フレームでラスタ済み、今フレームで再利用可 |
| Dirty | 何らかの invalidation で再ラスタ必要 |
| Allocated (NewlyMarked) | 今フレーム新規割当、未ラスタ |

### 2.2 Invalidation 経路

1. **インスタンス変形/移動**: `PrimitiveSceneProxy::HasDeformableMesh()` または `LocalToWorld` 変化 → 該当インスタンスが影響するページを dirty
2. **ライト移動 / ライト変化**: ライト全ページ dirty
3. **Nanite ストリーミング → LOD 不一致**: `r.Nanite.VSMInvalidateOnLODDelta = 1` で dirty
4. **CPU 側 culled ⇄ unculled 遷移**: revealed primitive invalidation

`r.Shadow.Virtual.Cache.InvalidateUseHZB = 1` で HZB 上で完全 occluded なインスタンスは invalidation スキップ（`VirtualShadowMapCacheManager.cpp:47-51`）。

### 2.3 静的 / 動的の分離

`r.Shadow.Virtual.Cache.FramesStaticThreshold = 100` フレーム動かないオブジェクトは **静的バケット**に昇格:
- 静的キャッシュは長期保持
- 動的キャッシュは短期、頻繁に dirty

`r.Shadow.Virtual.DynamicHZB = 1` で動的キャッシュ専用 HZB を構築（密な動的シーン用）。

### 2.4 ページ寿命

- 必要マーキング無しが N フレーム続いたら → free に戻す
- `r.Shadow.Virtual.Cache.MaxPageAgeSinceLastRequest = 1000` フレーム
- ライト単位寿命: `r.Shadow.Virtual.Cache.MaxLightAgeSinceLastRequest = 10` フレーム

### 2.5 GPU 駆動 invalidation

CPU 側 dirty tracking はインスタンス数で律速するので、GPU compute で並列処理:
- `VirtualShadowMapCacheGPUInvalidation.usf`: GPU 上でインスタンス AABB 変化 vs ページ AABB を判定
- `VirtualShadowMapCacheLoadBalancer.usf`: invalidation ワークを wave に分散

### 2.6 Deformable Mesh

スキニング / Spline Mesh / WPO は毎フレーム形状変化 → 自動 invalidation:

```cpp
// VirtualShadowMapCacheManager.cpp:53-58
static int32 GVSMCacheDeformableMeshesInvalidate = 1;
FAutoConsoleVariableRef CVarCacheDeformableMeshesInvalidate(
    TEXT("r.Shadow.Virtual.Cache.DeformableMeshesInvalidate"),
    GVSMCacheDeformableMeshesInvalidate,
    TEXT("If enabled, Primitive Proxies that are marked as having deformable meshes (HasDeformableMesh() == true) cause invalidations regardless of whether their transforms are updated."));
```

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `VirtualShadowMapCacheManager.cpp` | キャッシュ全体管理、CVar |
| `VirtualShadowMapCacheGPUInvalidation.usf` | GPU 駆動無効化 |
| `VirtualShadowMapCacheLoadBalancer.usf` | invalidation ワーク負荷分散 |
| `VirtualShadowMapPageMarking.usf` | レシーバから必要ページマーク |
| `VirtualShadowMapPageManagement.usf` | 割当 / 解放 |
| `VirtualShadowMapPhysicalPageManagement.usf` | 物理ページプール管理 |
| `VirtualShadowMapThrottle.usf` | フレーム予算超過時の throttle |

### 3.2 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.Shadow.Virtual.Cache` | 1 | キャッシュ機構 ON/OFF |
| `r.Shadow.Virtual.Cache.MaxPageAgeSinceLastRequest` | 1000 | 未要求ページの寿命（フレーム） |
| `r.Shadow.Virtual.Cache.MaxLightAgeSinceLastRequest` | 10 | 未参照ライトの寿命 |
| `r.Shadow.Virtual.Cache.FramesStaticThreshold` | 100 | 静的バケット昇格閾値 |
| `r.Shadow.Virtual.Cache.InvalidateUseHZB` | 1 | HZB ベース invalidation 抑制 |
| `r.Shadow.Virtual.Cache.DeformableMeshesInvalidate` | 1 | スキン/Spline/WPO 強制 invalidate |
| `r.Shadow.Virtual.Cache.DebugSkipRevealedPrimitivesInvalidation` | 0 | デバッグ: revealed invalidation スキップ |
| `r.Shadow.Virtual.AccumulateStats` | 0 | 統計を CSV に蓄積（デバッグ） |

### 3.3 Reserved Resource プール

```cpp
// VirtualShadowMapCacheManager.cpp:91-96
static TAutoConsoleVariable<int32> CVarVSMReservedResource(
    TEXT("r.Shadow.Virtual.AllocatePagePoolAsReservedResource"),
    1,
    TEXT("Allocate VSM page pool as a reserved/virtual texture, backed by N small physical memory allocations to reduce fragmentation."));
```

D3D12 Reserved Resource (TileMappings) で物理メモリ断片化抑制。

### 3.4 Throttle（予算制御）

`VirtualShadowMapThrottle.usf` がフレームあたり ラスタ可能ページ数を制限。超過分は次フレームに繰越（一時的に画質低下するが安定）。

---

## 4. 近似・省略の差分

| 項目 | 理想 | UE 実装 | 影響 |
|------|------|--------|------|
| Invalidation 精度 | ピクセル級 | ページ単位 + AABB | 同ページに動 / 静が混在 → 全 dirty |
| キャッシュ寿命 | 即解放 | N フレーム保持 | プール逼迫しないと残る |
| GPU 並列度 | 完全 | LoadBalancer 経由 | wave 不揃いで効率劣化（10-20%） |
| 動的/静的分離 | 完全 | 100 フレーム閾値 | 一時的に動いた静物がしばらく動的扱い |
| Reserved Resource | デバイス依存 | DX12 のみ最適 | 他 RHI で fragmentation |

---

## 5. パラメータと CVar

§3.2 にまとめ済み。`r.Shadow.Virtual.Cache=0` でキャッシュ無効化（デバッグ）。

---

## 6. 代替手法との比較

| 手法 | dirty 管理 | 採用 |
|------|----------|------|
| 全ページ毎フレーム再ラスタ | なし | デバッグモード |
| CPU dirty list | CPU AABB 比較 | UE4 旧シャドウ |
| **GPU 駆動 + HZB invalidation** | **GPU compute** | **UE5 VSM** |
| Sparse Virtual Texture (id Tech 5) | CPU streaming | テクスチャ系 |

---

## 7. 参考資料

- [S21] Karis 2021 §4 Caching
- 関連: [[vsm_overview]] / [[nanite_cluster_culling]]

---

## 8. 相談用フック

- **理解度チェック**:
  - HZB invalidation の動機 → 動いたが見えないインスタンスでも invalidate するのは無駄
  - 静的 / 動的バケット分離の理由 → 短期変化と長期キャッシュを別管理して dirty 連鎖を抑制
  - Throttle の意味 → 予算超過フレームでも安定 fps 維持
- **コード深掘り候補**:
  - `VirtualShadowMapCacheGPUInvalidation.usf` の AABB ⇄ ページマッピング
  - `VirtualShadowMapCacheLoadBalancer.usf` の wave 分散
  - DeformableMesh フラグ判定（Engine モジュール `PrimitiveSceneProxy`）
- **未読箇所**:
  - S21 §4.3 静的バケット昇格の閾値根拠
  - Reserved Resource の DX12 タイル管理詳細
  - Clipmap キャッシュ（カメラ移動時の toroidal）
- **次の派生**:
  - VSM 概要 → [[vsm_overview]]
  - PCF / PCSS フィルタ → [[shadow_pcf_pcss]]
  - Nanite との連携 → [[nanite_cluster_culling]]
