# リファレンス：LumenIrradianceFieldGather.cpp

- グループ: c - Tracing
- 上位: [[c_lumen_tracing]]
- 関連: [[ref_lumen_tracing_utils]] | [[ref_lumen_radiance_cache]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenIrradianceFieldGather.cpp`

---

## 概要

**実験的機能**（`r.Lumen.IrradianceFieldGather = 0` でデフォルト無効）。  
World Space に均等配置された照度プローブキャッシュ（Irradiance Field）を使い、  
Screen Probe Gather の代替として間接照明を提供する。  
`LumenRadianceCache` の仕組みを流用し、clipmap 構造でワールド空間を多重解像度でカバーする。

---

## グローバル変数

```cpp
// 実験的機能の有効フラグ（LumenTracingUtils.h で extern 宣言）
int32 GLumenIrradianceFieldGather = 0;
```

---

## LumenIrradianceFieldGather 名前空間

```cpp
namespace LumenIrradianceFieldGather {
    // Radiance Cache の入力パラメータを構築
    // clipmap 数・解像度・トレース設定を FRadianceCacheInputs にパック
    LumenRadianceCache::FRadianceCacheInputs SetupRadianceCacheInputs();
}
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.IrradianceFieldGather` | 0 | 機能の有効/無効（デフォルト無効・実験的）|
| `r.Lumen.IrradianceFieldGather.NumClipmaps` | 4 | Radiance Cache クリップマップ数 |
| `r.Lumen.IrradianceFieldGather.ClipmapWorldExtent` | 5000.0 | 最初のクリップマップの半径（cm）|
| `r.Lumen.IrradianceFieldGather.ClipmapDistributionBase` | 2.0 | クリップマップ間のサイズ比（Pow のベース）|
| `r.Lumen.IrradianceFieldGather.NumProbesToTraceBudget` | 100 | 1フレームに更新できるプローブ数 |
| `r.Lumen.IrradianceFieldGather.GridResolution` | 64 | クリップマップ内のプローブグリッド解像度 |
| `r.Lumen.IrradianceFieldGather.ProbeResolution` | 16 | 1プローブあたりのレイ数（16×16 = 256 レイ）|
| `r.Lumen.IrradianceFieldGather.IrradianceProbeResolution` | 6 | 照度プローブの 2D レイアウト解像度 |
| `r.Lumen.IrradianceFieldGather.OcclusionProbeResolution` | 16 | オクルージョンプローブの 2D レイアウト解像度 |
| `r.Lumen.IrradianceFieldGather.NumMipmaps` | 1 | Radiance Cache のミップマップ数 |
| `r.Lumen.IrradianceFieldGather.ProbeAtlasResolutionInProbes` | 128 | プローブアトラスの 1 辺あたりのプローブ数 |

---

## クリップマップ構造

```
NumClipmaps = 4 の場合:
  Clipmap 0: 半径 5000 cm  （近距離、高密度）
  Clipmap 1: 半径 10000 cm （ClipmapWorldExtent × 2^1）
  Clipmap 2: 半径 20000 cm （ClipmapWorldExtent × 2^2）
  Clipmap 3: 半径 40000 cm （ClipmapWorldExtent × 2^3）

各クリップマップ内:
  GridResolution = 64 → 64×64×64 = 262144 プローブ配置
  ※ Radiance Cache と共通の clipmap 更新メカニズムを使用
```

---

## Screen Probe Gather との比較

| 比較項目 | Screen Probe Gather（デフォルト）| Irradiance Field（実験的）|
|---------|------|------|
| プローブ配置 | スクリーン空間（適応的）| ワールド空間（均等グリッド）|
| 更新コスト | 毎フレーム全スクリーンを更新 | フレーム予算内で部分更新（100プローブ/フレーム）|
| 品質 | 高（視点依存の詳細）| 中（位置によらず一定品質）|
| レイテンシ | 低い | 高い（プローブ収束に複数フレーム）|
| 用途 | 一般的なリアルタイム GI | 大規模屋外シーン・VR 等 |

---

## RadianceCache との連携

```
LumenIrradianceFieldGather は LumenRadianceCache を内部で使用:

SetupRadianceCacheInputs()
  └─ FRadianceCacheInputs を構築
       │
       ├─ NumClipmaps = 4
       ├─ ClipmapWorldExtent = 5000 cm
       ├─ ProbeResolution = 16（16×16 = 256 レイ/プローブ）
       └─ NumProbesToTraceBudget = 100

RenderLumenIrradianceFieldGather()
  └─ LumenRadianceCache::RenderRadianceCache()
       → プローブを Surface Cache でトレース
       → SH（球面調和関数）に射影して保存
       → 補間で GBuffer ピクセルの照度を計算
```
