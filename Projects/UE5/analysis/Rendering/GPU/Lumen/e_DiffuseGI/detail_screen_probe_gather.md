# Lumen Screen Probe Gather シェーダー詳細

- グループ: e - Diffuse GI
- GPU 概要: [[01_gpu_overview]]
- CPU 詳細: [[e_lumen_diffuse_gi]]
- リファレンス: [[ref_screen_probe_gather]] | [[ref_restir_bent_normal]]

---

## 概要

Lumen Diffuse GI の中核。GBuffer のピクセルを代表する **Screen Probe** を配置し、  
各プローブからレイをトレースして **DiffuseIndirect テクスチャ**（全ピクセルの GI）を生成する。

```
GBuffer（深度・法線）
  ↓ [プローブ配置]
Screen Probe Grid（画面を 8×8 タイルに分割、各タイルに 1〜複数プローブ）
  ↓ [レイトレース]
LumenScreenProbeTracing.usf
  ├─ Screen Trace（HZB）         → 近距離
  ├─ Mesh SDF Trace             → 中距離
  └─ Global SDF / Radiance Cache → 遠距離
  ↓ [Gather + フィルタリング]
LumenScreenProbeGather.usf#ScreenProbeIntegrateCS
  ↓ [テンポラル蓄積]
LumenScreenProbeGatherTemporal.usf#ScreenProbeTemporalReprojectionCS
  ↓
RWDiffuseIndirect（Texture2DArray）
```

---

## パス構成詳細

### [1] プローブ配置（Placement）

```hlsl
// LumenScreenProbeGather.usf
ScreenProbeDownsampleDepthUniformCS()    // 均一グリッドのプローブ深度・法線をダウンサンプル
ScreenProbeAdaptivePlacementMarkCS()     // 高コントラスト領域に追加プローブをマーク
ScreenProbeAdaptivePlacementSpawnCS()    // 追加プローブを生成
```

### [2] Radiance Cache マーク

```hlsl
MarkRadianceProbesUsedByScreenProbesCS() // プローブが要求する Radiance Cache プローブをマーク
                                         // → Radiance Cache 更新バジェットに反映
```

### [3] レイ生成・トレース（LumenScreenProbeTracing.usf）

```hlsl
ClearTracesCS()                          // トレースバッファの初期化
ScreenProbeTraceScreenTexturesCS()       // Screen Trace（HZB 利用）
ScreenProbeCompactTracesCS()             // スクリーントレースミス分を Compact
ScreenProbeTraceMeshSDFsCS()             // Mesh SDF トレース
ScreenProbeTraceVoxelsCS()               // Global SDF / Voxel トレース
```

**3段階トレース戦略：**

```
レイ方向 → ScreenTrace (HZB) ──ヒット→ 高品質
                ↓ ミス
           MeshSDF Trace ──ヒット→ 精密
                ↓ ミス
           GlobalSDF / RadianceCache → 遠距離補完
```

### [4] Gather・タイル統合（LumenScreenProbeGather.usf）

```hlsl
InitScreenProbeTileIndirectArgsCS()      // Indirect Dispatch 引数設定
ScreenProbeTileClassificationBuildListsCS() // タイルを種別分類（均一/適応的）
ScreenProbeIntegrateCS()                 // プローブ輝度をスクリーン全ピクセルに展開
                                         // → RWDiffuseIndirect, RWRoughSpecularIndirect
```

### [5] テンポラル蓄積（LumenScreenProbeGatherTemporal.usf）

```hlsl
ScreenProbeTemporalReprojectionCS()      // 前フレームの履歴と現フレームをブレンド
                                         // → RWNewHistoryDiffuseIndirect（テンポラル安定化）
```

### [6] フィルタリング（LumenScreenProbeFiltering.usf）

空間フィルタ（スペーシャルフィルタ）でプローブ間の境界アーティファクトを除去。

---

## 入出力

### 入力

| リソース | 内容 |
|---------|------|
| `SceneDepth`, `SceneNormal` | プローブ配置の基礎 |
| `HierarchicalZBuffer` | Screen Trace のトレース加速 |
| `MeshSDFData` | Mesh SDF トレース用 |
| `GlobalSDF` | 遠距離 Voxel トレース用 |
| `FinalLightingAtlas` | ヒット点の輝度サンプリング元 |
| `RadianceCacheParameters` | 遠距離補完用 Radiance Cache 参照 |

### 出力

| テクスチャ | 内容 |
|---------|------|
| `RWDiffuseIndirect`（Texture2DArray[0]）| Diffuse 間接光（メイン GI）|
| `RWNewHistoryDiffuseIndirect`（Texture2DArray）| テンポラル履歴 |
| `RWRoughSpecularIndirect`（Texture2DArray）| 粗面反射の GI |

---

## CPU 呼び出しの流れ

```
RenderLumenFinalGather()                 // LumenScreenProbeGather.cpp:2094
  │
  ├─ PlaceScreenProbes()                 → ScreenProbeDownsampleDepthUniformCS 等
  ├─ MarkRadianceCacheProbesUsed()       → MarkRadianceProbesUsedByScreenProbesCS
  ├─ TraceScreenProbes()                 → LumenScreenProbeTracing.usf 各 CS
  ├─ [HW RT 時] LumenScreenProbeHardwareRayTracing.usf
  ├─ IntegrateScreenProbes()             → ScreenProbeIntegrateCS
  ├─ TemporalAccumulateScreenProbes()    → ScreenProbeTemporalReprojectionCS
  └─ FilterScreenProbes()                → LumenScreenProbeFiltering.usf
       ↓
  返値: FSSDSignalTextures（DiffuseIndirect/BackfaceDiffuse/Reflections のセット）
```

**CPU 側パラメーター割り当て（概略）：**

```cpp
// LumenScreenProbeGather.cpp 内
FLumenScreenProbeTracingShaderParameters TracingParams;
TracingParams.ScreenProbeParameters = GetScreenProbeParameters(GraphBuilder, View, ScreenProbeData);
TracingParams.MeshSDFGridParameters  = MeshSDFGridParameters;   // c グループでカリング済み
TracingParams.RadianceCacheParameters = RadianceCacheParams;    // d グループから伝搬
// → ScreenProbeTraceMeshSDFsCS 等に SHADER_PARAMETER_STRUCT_INCLUDE で渡す
```
