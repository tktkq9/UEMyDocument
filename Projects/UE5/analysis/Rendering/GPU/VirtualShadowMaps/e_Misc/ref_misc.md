# REF: VSM Misc シェーダー

- グループ: e - Misc
- 詳細: [[detail_misc]]
- CPU リファレンス: [[ref_vsm_array]] | [[ref_nanite_feedback]]
- ソース: `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapThrottle.usf`
          `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapCopyStats.usf`
          `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapPrintStats.usf`
          `Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapDebug.usf`

---

## VirtualShadowMapThrottle.usf

### ProcessPrevFramePerfDataCS

```hlsl
[numthreads(64, 1, 1)]
void ProcessPrevFramePerfDataCS(uint DispatchThreadId : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 前フレームの Nanite パフォーマンスフィードバック（HW/SW クラスター数）を現フレームのスロットルバッファに転送し、VSM ごとの統計と全体最大値を集計 |
| **入力** | `PrevNanitePerformanceFeedback`, `PrevThrottleBuffer`, `NextVirtualShadowMapData`, `NextVirtualShadowMapDataCount` |
| **出力** | `OutThrottleBuffer`（ヘッダー統計 + VSM ごとのエントリ）|
| **CPU 関数** | `FVirtualShadowMapArray::BuildPageAllocations()` |

### UpdateThrottleParametersCS

```hlsl
[numthreads(64, 1, 1)]
void UpdateThrottleParametersCS(uint DispatchThreadId : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | スロットルバッファのコスト統計を元に各 VSM の `ResolutionLodBias` を調整（過負荷時は解像度を下げる）|
| **入力** | `OutThrottleBuffer`（HW/SW クラスター統計）, `ThrottleMaxBiasLocal` |
| **出力** | `ProjectionData`（RWByteAddressBuffer）— `ResolutionLodBias` フィールドを更新 |
| **CPU 関数** | `FVirtualShadowMapArray::BuildPageAllocations()` |
| **コスト関数** | `CalculateCostHeuristic()`: HW × 0.00012176 + SW × 0.00002700 |

---

## VirtualShadowMapCopyStats.usf

### CopyStatsCS

```hlsl
[numthreads(1, 1, 1)]
void CopyStatsCS()
```

| 項目 | 内容 |
|-----|------|
| **目的** | GPU 統計バッファを CPU 読み取り可能なバッファにコピー（パフォーマンスカウンター転送）|
| **入力** | `StatsBuffer`（GPU 統計カウンター）|
| **出力** | `OutStatsBuffer`（CPU 可読コピー）|
| **CPU 関数** | `FVirtualShadowMapArray::PostRender()` |

---

## VirtualShadowMapPrintStats.usf

### LogVirtualSmStatsCS

```hlsl
[numthreads(VSM_GENERATE_STATS_GROUPSIZE, 1, 1)]  // VSM_GENERATE_STATS=1 時
[numthreads(1, 1, 1)]                              // それ以外
void LogVirtualSmStatsCS(uint ThreadId : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | `r.Shadow.Virtual.ShowStats` が有効な場合、VSM の統計情報を画面のシェーダーデバッグプリントで表示 |
| **入力** | `StatsBuffer`, `StatusMessageId`, `StatsMessageId` |
| **出力** | `FShaderPrintContext` 経由での画面テキスト出力 |
| **CPU 関数** | `FVirtualShadowMapArray::PostRender()` の `LogStats()` |
| **VSM_GENERATE_STATS** | 有効時のみ詳細統計を収集。ThreadId=0 のみが `SendStatusMessage()` / `SendStatsMessage()` を呼ぶ |

#### 内部ヘルパー

| 関数 | 役割 |
|-----|------|
| `SendStatusMessage()` | VSM の状態メッセージ（ページ数、解像度）を出力 |
| `SendStatsMessage()` | 詳細統計（キャッシュ率、無効化数、クラスター数）を出力 |
| `BuildNaniteGeoHistogram()` | Nanite クラスター数のヒストグラムを構築 |
| `PrintLeftAlign()` | 左寄せフォーマット出力（uint / float のオーバーロードあり）|
| `PrintUnits()` | 数値を単位付きで出力（K / M 等のスケール調整）|

---

## VirtualShadowMapDebug.usf

### DebugVisualizeVirtualSmCS

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_XY, VSM_DEFAULT_CS_GROUP_XY, 1)]
void DebugVisualizeVirtualSmCS(uint2 PixelPos : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | `VisualizeModeId` に応じたデバッグビジュアライゼーションを `OutVisualize` テクスチャに書き込む |
| **入力** | `VisualizeModeId`, `VirtualShadowMapId`, `DebugTargetWidth/Height`, VSM 各種テクスチャ |
| **出力** | `OutVisualize`（RGBA float4）— `Output.a > 0` の場合のみ書き込み |
| **CPU 関数** | `FVirtualShadowMapArray::AddDebugVisualizationPasses()` |

#### VisualizeModeId 一覧

| ID | モード | 内容 |
|----|--------|------|
| `VIRTUAL_SHADOW_MAP_VISUALIZE_CLIPMAP_VIRTUAL_SPACE` | クリップマップ仮想空間 | クリップマップのページレイアウトをカラーマップで表示 |
| `VIRTUAL_SHADOW_MAP_VISUALIZE_SMRT_RAY_COUNT` | SMRT レイ数 | SMRT のサンプル数をヒートマップで表示（`TonemapProjectionDebugTexturePS` と連携）|
| その他 | ページ状態・物理アトラス等 | `VirtualShadowMapVisualize.ush` に定義された追加モード |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Shadow.Virtual.ShowStats` | 0 | 1=統計をオーバーレイ表示（`LogVirtualSmStatsCS`）|
| `r.Shadow.Virtual.Throttling` | 1 | 解像度スロットリングを有効化（`UpdateThrottleParametersCS`）|
| `r.Shadow.Virtual.ResolutionLodBiasMax` | — | スロットリングで適用する最大 LOD バイアス |

---

> [!note]- Throttle の LOD バイアスが解像度に影響する仕組み
> VSM の各 Shadow Map は `FVirtualShadowMapProjectionShaderData::ResolutionLodBias` フィールドを持つ。  
> `ApplyResolutionLODBias()` がこの値をインクリメントすると、ページマーキング時の `GetBiasedClipmapLevel()` が  
> より粗い MIP レベルを選択するようになり、要求ページ数（=描画コスト）が削減される。  
> バイアスが減少（回復フェーズ）するとより細かい MIP が選択されて品質が戻る。

> [!note]- CopyStatsCS が必要な理由
> GPU 統計カウンターは描画フレーム中にインクリメントされる RWBuffer にある。  
> CPU は RHI の Map/Unmap でこれを読めるが、直接 Map すると GPU と CPU が同期して  
> パイプラインストールが発生する。`CopyStatsCS` でステージングバッファにコピーしておくことで、  
> 数フレーム後に非同期で CPU が読み取れる（レイテンシはあるが GPU を止めない）。

> [!note]- LogVirtualSmStatsCS のスレッドグループサイズの二段階
> `VSM_GENERATE_STATS=1` のとき `VSM_GENERATE_STATS_GROUPSIZE` スレッドで起動して並列統計収集を行い、  
> ThreadId=0 のみが最終的な表示処理（`SendStatsMessage`）を行う。  
> `VSM_GENERATE_STATS=0` のとき `[numthreads(1,1,1)]` で ThreadId=0 のみが起動し、  
> ステータスメッセージ（`SendStatusMessage`）のみを出力する軽量版になる。
