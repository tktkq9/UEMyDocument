# VSM Misc シェーダー詳細

- グループ: e - Misc
- GPU 概要: [[01_vsm_gpu_overview]]
- CPU 詳細: [[a_vsm_array]] | [[f_nanite_debug_editor]]
- リファレンス: [[ref_misc]]

---

## 概要

VSM のメインパイプライン外で実行される補助シェーダー群。  
パフォーマンス適応（Throttle）、統計収集（Stats/CopyStats）、デバッグビジュアライゼーション（Debug）を担当する。

---

## Throttle（VirtualShadowMapThrottle.usf）

### 目的

前フレームの Nanite パフォーマンスデータ（HW/SW クラスター数）を元に、  
VSM の解像度 LOD バイアスを動的に調整することで GPU 過負荷を防ぐ。

### ProcessPrevFramePerfDataCS — パフォーマンスデータ転送

```hlsl
[numthreads(64, 1, 1)]
void ProcessPrevFramePerfDataCS(uint DispatchThreadId : SV_DispatchThreadID)
```

前フレームの Nanite パフォーマンスフィードバック（`PrevNanitePerformanceFeedback`）を読み取り、  
現フレームのスロットルバッファ（`OutThrottleBuffer`）に転送する。

```hlsl
// ヘッダー（全体統計）
OutThrottleBuffer[VSM_TB_HEADER_TOTAL_HW_CLUSTERS] = TotalClustersHW;
OutThrottleBuffer[VSM_TB_HEADER_TOTAL_SW_CLUSTERS] = TotalClustersSW;

// エントリ（VSM ごとの統計）
// 前フレームの VSM ID → 現フレームの VSM ID にマッピング（NextVSMData 経由）
OutThrottleBuffer[...VSM_TB_ENTRY_HW_CLUSTERS] = ClustersHW;
OutThrottleBuffer[...VSM_TB_ENTRY_SW_CLUSTERS] = ClustersSW;

// 最大値を InterlockedMax で集計
InterlockedMax(OutThrottleBuffer[VSM_TB_HEADER_MAX_HW_CLUSTERS], ClustersHW);
```

### UpdateThrottleParametersCS — LOD バイアス更新

```hlsl
[numthreads(64, 1, 1)]
void UpdateThrottleParametersCS(uint DispatchThreadId : SV_DispatchThreadID)
```

コストヒューリスティック（`CalculateCostHeuristic`）でフレームコストを推定し、  
コストが閾値を超えた場合に各 VSM の `ResolutionLodBias` を増加させて解像度を下げる。

```hlsl
float CalculateCostHeuristic(float NaniteClustersHW, float NaniteClustersSW)
{
    // HW クラスターと SW クラスターのコスト係数が異なる
    return 0.00012176f * NaniteClustersHW + 0.00002700f * NaniteClustersSW;
}

// コストが ThrottleMaxBias 超 → ApplyResolutionLODBias() で各 VSM のバイアスを増加
// コストが閾値以下 → バイアスを減少（徐々に品質を回復）
```

`ApplyResolutionLODBias()` インライン関数が `ProjectionData.ResolutionLodBias` を  
`RWByteAddressBuffer` に直接書き込む。

---

## CopyStats（VirtualShadowMapCopyStats.usf）

```hlsl
[numthreads(1, 1, 1)]
void CopyStatsCS()
```

統計バッファを CPU 読み取り可能なバッファにコピーする。  
パフォーマンスカウンター（ページ数、キャッシュヒット率など）を GPU → CPU に転送。

---

## PrintStats / LogStats（VirtualShadowMapPrintStats.usf）

```hlsl
[numthreads(VSM_GENERATE_STATS_GROUPSIZE, 1, 1)] // または [numthreads(1, 1, 1)]
void LogVirtualSmStatsCS(uint ThreadId : SV_DispatchThreadID)
```

`VSM_GENERATE_STATS=1` 時に統計情報を画面上のシェーダーデバッグプリント（`FShaderPrintContext`）で表示する。  
ThreadId=0 のみが `SendStatusMessage()` / `SendStatsMessage()` を呼び出す。

表示される統計情報：
- 割り当て済みページ数（物理 / 仮想）
- キャッシュ済み / 未キャッシュページ数
- 無効化されたページ数
- Nanite クラスター数（HW / SW 別）
- Nanite ジオメトリヒストグラム（`BuildNaniteGeoHistogram`）

---

## Debug（VirtualShadowMapDebug.usf）

```hlsl
[numthreads(VSM_DEFAULT_CS_GROUP_XY, VSM_DEFAULT_CS_GROUP_XY, 1)]
void DebugVisualizeVirtualSmCS(uint2 PixelPos : SV_DispatchThreadID)
```

`VisualizeModeId` に応じてデバッグビジュアライゼーションを `OutVisualize` テクスチャに書き込む。

| VisualizeModeId | 表示内容 |
|----------------|---------|
| `VIRTUAL_SHADOW_MAP_VISUALIZE_CLIPMAP_VIRTUAL_SPACE` | クリップマップの仮想空間レイアウト |
| `VIRTUAL_SHADOW_MAP_VISUALIZE_SMRT_RAY_COUNT` | SMRT レイ数のヒートマップ |
| その他 | 物理ページ占有率・キャッシュ状態等（`VirtualShadowMapVisualize.ush` に定義）|

`Output.a > 0` の場合のみ書き込み（他のパスが書いたピクセルを上書きしない）。

---

## 入出力

### Throttle

| リソース | 内容 |
|---------|------|
| `PrevNanitePerformanceFeedback` | 前フレームの Nanite HW/SW クラスター数（読み取り）|
| `OutThrottleBuffer` | 現フレーム用スロットルバッファ（書き込み）|
| `PrevThrottleBuffer` | 前フレームのスロットルバッファ（LOD バイアス継続用）|
| `ProjectionData`（RW）| VSM ごとの `ResolutionLodBias`（`UpdateThrottleParametersCS` が書き込み）|
| `NextVirtualShadowMapData` | 前→現フレームの VSM ID マッピング |

### Stats / Debug

| リソース | 内容 |
|---------|------|
| `StatsBuffer` | VSM 統計カウンター（ページ数 / キャッシュ率等）|
| `OutStatsBuffer` | CPU 読み取り用にコピーした統計バッファ |
| `OutVisualize` | デバッグビジュアライゼーション出力テクスチャ |

---

## CPU 呼び出しの流れ

```
FVirtualShadowMapArray::BuildPageAllocations() 後半
  └─ ProcessPrevFramePerfDataCS
  └─ UpdateThrottleParametersCS

FVirtualShadowMapArray::PostRender()           // VirtualShadowMapArray.cpp:1729
  └─ CopyStatsCS
  └─ LogVirtualSmStatsCS（r.Shadow.Virtual.ShowStats 時）

FVirtualShadowMapArray::AddDebugVisualizationPasses()
  └─ DebugVisualizeVirtualSmCS
```
