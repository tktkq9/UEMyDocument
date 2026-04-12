# REF: Lumen ReSTIR Gather / Screen Space Bent Normal シェーダー

- グループ: e - Diffuse GI
- 詳細: [[detail_screen_probe_gather]]
- CPU リファレンス: [[ref_lumen_diffuse_indirect]]
- ソース: `Engine/Shaders/Private/Lumen/LumenReSTIRGather.usf`  
          `Engine/Shaders/Private/Lumen/LumenScreenSpaceBentNormal.usf`  
          `Engine/Shaders/Private/Lumen/LumenShortRangeAOHardwareRayTracing.usf`

---

## LumenReSTIRGather.usf

### 概要

**ReSTIR（Reservoir-based SpatioTemporal Importance Resampling）** を使った  
より高品質な Diffuse GI 計算（`r.Lumen.ScreenProbeGather.ReSTIR=1` 時に有効）。  
通常の Screen Probe Gather に置き換わる上位互換パス。

### 主要エントリポイント（詳細はソース参照）

| エントリポイント | 役割 |
|---------------|------|
| `GenerateInitialSamplesCS()` | 各プローブの初期候補サンプル（レイ）を生成 |
| `TemporalResamplingCS()` | 前フレームのリザーバとブレンドして候補を絞り込み |
| `SpatialResamplingCS()` | 隣接プローブのリザーバから候補を再サンプリング |
| `EvaluateFinalSamplesCS()` | 最終候補を評価してトレース実行 |
| `IntegrateReSTIRSamplesCS()` | 結果をスクリーン全ピクセルに展開 |

### CPU 呼び出し

```cpp
// LumenScreenProbeGather.cpp
if (UseReSTIRGather(View))
{
    // 通常の RenderLumenScreenProbeGather() の代わりに呼ばれる
    RenderLumenReSTIRGather(GraphBuilder, SceneTextures, ...);
}
```

---

## LumenScreenSpaceBentNormal.usf

### 概要

**Screen Space Bent Normal**：スクリーンスペースのレイキャストで各ピクセルの「遮蔽方向」を計算し、  
AO をベクトルとして表現する（短距離の自己遮蔽を正確に再現）。  
DiffuseIndirectComposite の `FScreenBentNormalMode` パーミュテーションで使用される。

### LumenScreenSpaceBentNormalCS

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, 1)]
void LumenScreenSpaceBentNormalCS(uint3 DispatchThreadId : SV_DispatchThreadID)
```

| 項目 | 内容 |
|-----|------|
| **目的** | 各ピクセルから複数のスクリーンレイを投射して遮蔽方向（Bent Normal）を推定 |
| **入力** | `SceneDepth`, `SceneNormal`, `HierarchicalZBuffer` |
| **出力** | `RWScreenBentNormal`（float3 Bent Normal ベクトル + AO スカラー）|
| **CPU 関数** | `RenderLumenScreenSpaceBentNormal()` |

#### パーミュテーション

| クラス | 説明 |
|-------|------|
| `SHORT_RANGE_AO_SUPPORTED` | Substrate タイルパーミュテーション |
| `INDIRECT_DISPATCH` | Indirect Dispatch 形式 |

---

## LumenShortRangeAOHardwareRayTracing.usf

### 概要

`r.Lumen.ScreenProbeGather.ShortRangeAO.HardwareRayTracing=1` 時に  
Screen Space Bent Normal を **HW RT で代替**するシェーダー。  
スクリーン外の遮蔽も計算できるため精度が高い。

```hlsl
RAY_TRACING_ENTRY_RAYGEN(LumenShortRangeAOHardwareRayTracingRGS)
```

| 目的 | HW RT で近距離の AO レイをトレースして Bent Normal を生成 |
|-----|------|
| CPU 関数 | `DispatchLumenShortRangeAOHardwareRayTracing()` |

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.Lumen.ScreenProbeGather.ReSTIR` | 0 | ReSTIR Gather を使うか（高品質・高コスト）|
| `r.Lumen.ScreenProbeGather.ShortRangeAO` | 1 | Screen Space Bent Normal AO を有効化 |
| `r.Lumen.ScreenProbeGather.ShortRangeAO.NumRaysPerPixel` | 4 | Bent Normal 推定のレイ数 |
| `r.Lumen.ScreenProbeGather.ShortRangeAO.HardwareRayTracing` | 0 | HW RT 版を使うか |

---

> [!note]- Bent Normal が GI 品質を上げる理由
> 通常の SSAO はスカラー値で「遮蔽されているか否か」を表現するだけ。  
> Bent Normal はスカラーに加えて「最も開いている方向（Bent Direction）」ベクトルを持つ。  
> DiffuseIndirectComposite の `FDiffuseIndirectCompositePS` はこの方向を使い、GI の Irradiance を Bent Normal 方向でバイアスさせることで、暗い隅で GI が浮く（Light Leaking）問題を抑える。

> [!note]- ReSTIR と通常 Screen Probe Gather の切り替え
> `UseReSTIRGather()` が true の場合、`RenderLumenFinalGather()` は `RenderLumenReSTIRGather()` を呼ぶ（`LumenScreenProbeGather.cpp:2094`）。  
> ReSTIR は同じ Screen Probe グリッドを使うが、リザーバ（候補リスト）を時間・空間方向に再サンプリングすることで同じレイ数でより分散の少ない GI を得る。  
> デフォルト無効（コスト増加のため）だが、Ray Budget に余裕がある高品質モードで有効化する。
