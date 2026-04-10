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

### 使用箇所

- [[ref_lumen_tracing_utils]] — `extern int32 GLumenIrradianceFieldGather` として宣言
- [[ref_lumen_diffuse_indirect]] — `RenderLumenDiffuseIndirect()` で GI 方式の選択分岐に使用

---

## LumenIrradianceFieldGather::SetupRadianceCacheInputs

```cpp
namespace LumenIrradianceFieldGather {
    LumenRadianceCache::FRadianceCacheInputs SetupRadianceCacheInputs();
}
```

### 戻り値

`LumenRadianceCache::FRadianceCacheInputs` — Radiance Cache の動作設定をまとめた構造体。

### FRadianceCacheInputs の主要フィールド

| フィールド | 型 | 値（Irradiance Field 設定）| 説明 |
|-----------|-----|--------------------------|------|
| `NumClipmaps` | `int32` | 4（CVar）| クリップマップ数 |
| `ClipmapWorldExtent` | `float` | 5000.0（CVar, cm）| 最初のクリップマップの半径 |
| `ClipmapDistributionBase` | `float` | 2.0（CVar）| クリップマップ間のサイズ比 |
| `ProbeResolution` | `int32` | 16（CVar）| 1 プローブあたりのレイ数（16×16=256）|
| `NumProbesToTraceBudget` | `int32` | 100（CVar）| 1 フレームあたりの更新プローブ数 |
| `GridResolution` | `int32` | 64（CVar）| クリップマップ内のプローブグリッド解像度（64³）|
| `IrradianceProbeResolution` | `int32` | 6（CVar）| 照度プローブの 2D レイアウト解像度 |
| `OcclusionProbeResolution` | `int32` | 16（CVar）| オクルージョンプローブの 2D レイアウト解像度 |
| `NumMipmaps` | `int32` | 1（CVar）| Radiance Cache のミップマップ数 |
| `ProbeAtlasResolutionInProbes` | `int32` | 128（CVar）| プローブアトラスの 1 辺あたりのプローブ数 |

### 内部処理フロー

```cpp
LumenRadianceCache::FRadianceCacheInputs SetupRadianceCacheInputs() {
    LumenRadianceCache::FRadianceCacheInputs Inputs;

    // 1. クリップマップ構造の設定
    Inputs.NumClipmaps = GLumenIrradianceFieldGatherNumClipmaps; // 4
    Inputs.ClipmapWorldExtent = GLumenIrradianceFieldGatherClipmapWorldExtent; // 5000 cm

    // 2. プローブ解像度の設定
    Inputs.ProbeResolution = GLumenIrradianceFieldGatherProbeResolution; // 16
    Inputs.GridResolution  = GLumenIrradianceFieldGatherGridResolution;  // 64

    // 3. 更新予算の設定
    Inputs.NumProbesToTraceBudget = GLumenIrradianceFieldGatherNumProbesToTraceBudget; // 100

    // 4. アトラス解像度の設定
    Inputs.ProbeAtlasResolutionInProbes = GLumenIrradianceFieldGatherProbeAtlasResolutionInProbes;

    return Inputs;
}
```

### 使用箇所

- [[ref_lumen_irradiance_field]] — `RenderLumenIrradianceFieldGather()` の冒頭で呼ばれ、RadianceCache に入力として渡される
- [[ref_lumen_radiance_cache]] — `LumenRadianceCache::RenderRadianceCache()` の引数 `FRadianceCacheInputs` として使用

---

## RenderLumenIrradianceFieldGather

```cpp
void RenderLumenIrradianceFieldGather(
    FRDGBuilder& GraphBuilder,
    const FSceneTextures& SceneTextures,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    TArray<FViewInfo>& Views,
    FGlobalIlluminationPluginResources& GlobalIlluminationPluginResources);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `SceneTextures` | `const FSceneTextures&` | GBuffer テクスチャ参照 |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | フレームアトラスバッファ参照 |
| `Views` | `TArray<FViewInfo>&` | ビュー配列（通常は 1 要素）|
| `GlobalIlluminationPluginResources` | `FGlobalIlluminationPluginResources&` | GI 出力テクスチャ（後続パスへの受け渡し用）|

### 内部処理フロー

1. **有効チェック**
   ```cpp
   if (!GLumenIrradianceFieldGather) { return; }
   ```

2. **Radiance Cache 入力構築**
   ```cpp
   LumenRadianceCache::FRadianceCacheInputs RadianceCacheInputs =
       LumenIrradianceFieldGather::SetupRadianceCacheInputs();
   ```

3. **Radiance Cache のトレース・更新**
   ```cpp
   // LumenRadianceCache の更新パイプラインを流用
   LumenRadianceCache::RenderRadianceCache(
       GraphBuilder, SceneTextures, FrameTemporaries,
       Views[0], RadianceCacheInputs, ...);
   // → プローブを Surface Cache でトレース
   // → 結果を SH（球面調和関数）に射影して保存
   ```

4. **GBuffer ピクセルへの照度補間**
   ```cpp
   // Radiance Cache から補間して最終的な Diffuse Indirect を計算
   RenderLumenIrradianceFieldGatherInterpolation(GraphBuilder, SceneTextures, Views[0], ...);
   // → GlobalIlluminationPluginResources.DiffuseIndirect に出力
   ```

### 使用箇所

- [[ref_lumen_diffuse_indirect]] — `GLumenIrradianceFieldGather != 0` の場合に `RenderLumenDiffuseIndirect()` から呼ばれる（Screen Probe Gather の代替）

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.IrradianceFieldGather` | 0 | 機能の有効/無効（デフォルト無効・実験的）|
| `r.Lumen.IrradianceFieldGather.NumClipmaps` | 4 | Radiance Cache クリップマップ数 |
| `r.Lumen.IrradianceFieldGather.ClipmapWorldExtent` | 5000.0 | 最初のクリップマップの半径（cm）|
| `r.Lumen.IrradianceFieldGather.ClipmapDistributionBase` | 2.0 | クリップマップ間のサイズ比（2^n で拡大）|
| `r.Lumen.IrradianceFieldGather.NumProbesToTraceBudget` | 100 | 1 フレームに更新できるプローブ数 |
| `r.Lumen.IrradianceFieldGather.GridResolution` | 64 | クリップマップ内のプローブグリッド解像度 |
| `r.Lumen.IrradianceFieldGather.ProbeResolution` | 16 | 1 プローブあたりのレイ数（16×16 = 256 レイ）|
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
  ※ カメラ移動によってプローブが無効化・再更新される
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
