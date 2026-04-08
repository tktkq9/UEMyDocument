# リファレンス：LumenRadiosity.h / LumenRadiosity.cpp

- グループ: b - Scene Lighting
- 上位: [[b_lumen_scene_lighting]]
- 関連: [[ref_lumen_scene_lighting]] | [[ref_lumen_radiance_cache]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenRadiosity.h/cpp`

---

## 概要

Surface Cache 間の**マルチバウンス拡散 GI**（間接照明）を計算するシステム。  
各 Card 表面に均等間隔で**放射照度プローブ**（Radiosity Probe）を配置し、  
Surface Cache の Radiance をトレースして SH（球面調和関数）に焼き付ける。  
計算結果は Indirect Lighting アトラスに書き込まれる。

---

## LumenRadiosity 名前空間

### FFrameTemporaries

1 フレームの Radiosity 計算に必要な一時データを保持する構造体。  
`InitFrameTemporaries()` で初期化され、レンダリング全体に渡される。

```cpp
namespace LumenRadiosity {
    struct FFrameTemporaries {
        // --- フレーム状態 ---
        bool bIndirectLightingHistoryValid; // 前フレームの履歴が有効か（テンポラル再利用用）
        bool bUseProbeOcclusion;            // プローブオクルージョンを使うか

        // --- プローブ設定 ---
        int32 ProbeSpacing;              // プローブ間隔（Surface Cache テクセル単位）
        int32 HemisphereProbeResolution; // 半球1辺あたりのトレース数

        // --- アトラスサイズ ---
        FIntPoint ProbeAtlasSize;        // SH 結果アトラスのサイズ
        FIntPoint ProbeTracingAtlasSize; // トレース用アトラスのサイズ

        // --- RDG テクスチャ ---
        FRDGTextureRef TraceRadianceAtlas;    // トレースした Radiance（HDR）
        FRDGTextureRef TraceHitDistanceAtlas; // トレースのヒット距離（オクルージョン用）

        // --- SH アトラス（RGB 分離）---
        FRDGTextureRef ProbeSHRedAtlas;   // SH 係数 (R チャンネル)
        FRDGTextureRef ProbeSHGreenAtlas; // SH 係数 (G チャンネル)
        FRDGTextureRef ProbeSHBlueAtlas;  // SH 係数 (B チャンネル)
    };

    // Radiosity が有効かどうかの判定
    bool IsEnabled(const FSceneViewFamily& ViewFamily);

    // FFrameTemporaries を初期化（プローブ設定・アトラス確保）
    void InitFrameTemporaries(
        FRDGBuilder& GraphBuilder,
        const FLumenSceneFrameTemporaries& LumenFrameTemporaries,
        const FViewInfo& View,
        FFrameTemporaries& OutFrameTemporaries);

    // Surface Cache のダウンサンプル係数（プローブアトラスはダウンサンプル済み）
    uint32 GetAtlasDownsampleFactor();
}
```

---

## 主要 CVar（LumenRadiosity.cpp）

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.Radiosity` | 1 | Radiosity の有効/無効（0 で無効化） |
| `r.LumenScene.Radiosity.ProbeSpacing` | 4 | プローブ間隔（Surface Cache テクセル単位）|
| `r.LumenScene.Radiosity.HemisphereProbeResolution` | 4 | 半球1辺あたりのトレース数（4×4 = 16 レイ）|
| `r.LumenScene.Radiosity.SpatialFilterProbes` | 1 | プローブの空間フィルタリングを行うか |
| `r.LumenScene.Radiosity.SpatialFilterProbes.KernelSize` | 1 | フィルタカーネルサイズ（大きいほどノイズ減・リーク増）|
| `r.LumenScene.Radiosity.ProbePlaneWeighting` | 1 | 平面距離による重みづけでリークを抑制 |
| `r.LumenScene.Radiosity.ProbeOcclusion` | 0 | プローブ補間時に深度テストするか（SW RT では精度不足で無効推奨）|
| `r.LumenScene.Radiosity.ProbeOcclusionStrength` | 0.5 | プローブオクルージョン強度（0〜1）|
| `r.LumenScene.Radiosity.SpatialFilterProbes.PlaneWeightingDepthScale` | -100.0 | 平面重みの深度スケール（絶対値大きいほど厳格）|
| `r.LumenScene.Radiosity.MaxRayIntensity` | 40.0 | トレース輝度クランプ値（露光相対）|
| `r.LumenScene.Radiosity.DistanceFieldSurfaceBias` | 10.0 | SDF トレースの表面バイアス（cm）|
| `r.LumenScene.Radiosity.DistanceFieldSurfaceSlopeBias` | 5.0 | SDF トレースの法線方向バイアス |
| `r.LumenScene.Radiosity.HardwareRayTracing.SurfaceBias` | 0.1 | HW RT トレースの表面バイアス |
| `r.LumenScene.Radiosity.UpdateFactor` | 64 | 更新レート（全ページの 1/64 を毎フレーム更新）|

---

## Radiosity プローブの仕組み

### プローブ配置

```
Surface Cache Card (例: 128×128 テクセル)
  ProbeSpacing = 4 の場合:
  → 32×32 個のプローブを均等配置
  → 各プローブは半球 4×4 = 16 方向にトレース
```

### SH（球面調和関数）への変換

```
各プローブのトレース結果（Radiance × 方向）
  → L1 SH（4係数 × RGB = 12 値）に射影
  → ProbeSHRed / Green / Blue アトラスに保存
  → 補間時に Card テクセルの法線方向で SH を評価 → Irradiance
```

---

## Radiosity 処理フロー

```
RenderLumenRadiosity(CardUpdateContext)
  │
  ├─ InitFrameTemporaries()     ← プローブアトラスサイズ計算・RDG テクスチャ確保
  │
  ├─ [トレースパス]
  │   ├─ SDF Tracing  : Surface Cache を SDF でトレース → TraceRadianceAtlas
  │   └─ HW RT Tracing: RT が有効な場合は HW RT でトレース（高精度）
  │
  ├─ [フィルタリング]
  │   ├─ SpatialFilterProbes: 近傍プローブから SH を空間フィルタ
  │   └─ ProbePlaneWeighting: 平面距離による重みで裏面プローブを除外
  │
  ├─ ProjectToSH()              ← Radiance → SH 係数に変換・蓄積
  │
  └─ InterpolateProbes()        ← SH を Card テクセルに補間展開
        → Indirect Lighting アトラスに書き込み
```

---

## UpdateFactor の意味

```cpp
// r.LumenScene.Radiosity.UpdateFactor = 64 の場合:
// 全 Card ページのうち 1/64 ずつを毎フレーム更新
// 64 フレームで全ページが更新される（1フレームの負荷を分散）
//
// Direct Lighting UpdateFactor = 32 より遅い理由:
// Radiosity はバウンス光なので Direct Lighting より変化が緩やか
```
