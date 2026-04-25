# Lumen Radiance Cache（球面プローブベースの遠距離放射輝度キャッシュ）

- 上位: [[_algorithm_index]]
- 関連: [[lumen_surface_cache]] / [[lumen_final_gather]] / [[lumen_sw_rt]] / [[lumen_hw_rt]]
- 採用システム: Lumen ScreenProbeGather の遠方サンプル, Translucency Volume Lighting, Volumetric Cloud GI
- 出典:
  - **S10**: Wright / Heitz / Hillaire 2022 SIGGRAPH "Lumen: Real-time Global Illumination in Unreal Engine 5" — §5 World-space Radiance Cache
  - 影響元: Sloan 2008 Stupid Spherical Harmonics Tricks（SH 球面表現の系譜）

---

## 1. 何のためのアルゴリズムか

ScreenProbeGather のレイは **画面上のサンプル点から放たれる** が、そのレイが遠くまで飛ぶと:
- 各レイで毎フレーム全長トレースするのは重い
- 遠方の照明は **空間的に低周波** で、近隣の他レイと結果がほぼ同じ → **キャッシュで使い回し可能**

Radiance Cache は **ワールド空間に配置した球面プローブ** に放射輝度を貯め、レイが一定距離（プローブ半径 TMin）を越えたら **プローブを参照して打ち切る**。

### 素朴な手法の問題

- **全 ScreenProbe レイを毎回フルトレース**: GPU 時間が距離に比例
- **Static Light Probe のみ**: 動的光源・動的シーンで破綻
- **Voxel Cone Tracing**: 解像度を上げると VRAM 爆発、leaking

### Lumen Radiance Cache の貢献

- **Sparse 3D グリッドプローブ**（Clipmap 階層、典型 4 段、近: 高密度 / 遠: 疎）
- 各プローブは **球面オクタヘドラルマップ（典型 32×32）** で全方位放射輝度を保持
- ScreenProbe レイが TMin（典型 1〜数 m）を越えたら Radiance Cache をサンプリング → トレース打ち切り
- プローブ単位で **時系列再利用**（更新は priority queue で予算配分）

---

## 2. 理論

### 2.1 階層クリップマップ

ビュー原点中心に N 段（典型 4）の同心立方体グリッド:

| Clipmap | セルサイズ | 範囲 |
|---------|----------|------|
| 0 | 50 cm | カメラ近傍 |
| 1 | 1 m | 中距離 |
| 2 | 4 m | 遠距離 |
| 3 | 16 m | 超遠距離 |

各クリップマップは `RadianceProbeClipmapResolution³`（典型 48³）の **インダイレクションテクスチャ** を持ち、有効プローブのみ物理アトラスにアロケート。

### 2.2 プローブ表現

- 1 プローブ = 球面方向の放射輝度関数
- UE では **オクタヘドラルマップ**（球面 → 2D 正方形に展開）で `RadianceProbeResolution × RadianceProbeResolution` テクセル（典型 32×32）
- アトラスは `ProbeAtlasResolutionInProbes × FinalProbeResolution`（典型 128×128 プローブ × 32 = 4096²）

### 2.3 プローブ更新パイプライン

1. **Mark**: ScreenProbeGather の各レイが「このプローブを参照する」とマーク（compute pass `MarkRadianceProbesUsedByVisualizeCS` 等）
2. **Allocate**: 使用された未割当プローブに物理アトラスのスロットを割当（フリーリストから pop）
3. **Trace Tile 生成**: プローブごとに更新する方向（trace tile）を BRDF 重要度で決定
4. **Trace**: SW/HW RT でレイを飛ばし、ヒット先 Surface Cache から放射輝度取得
5. **Spatial Filter**（オプション、`r.Lumen.RadianceCache.SpatialFilterProbes = 1`）: 隣接プローブと角度制限付き平均
6. **Build Final Atlas**: ミップ生成、border 追加でバイリニア対応

### 2.4 重要度サンプリング（Trace Tile BRDF Importance）

- プローブの全 32×32 = 1024 方向を毎フレーム更新するのは重い
- **Priority Histogram**（16 buckets）で「どの方向が今フレーム重要か」を集計（ScreenProbe レイの集計から）
- 上位バケットを supersample（フル解像度）、下位を downsample（半解像度）→ 予算内で品質最適化

`r.Lumen.RadianceCache.SupersampleTileBRDFThreshold = 0.1` が判定閾値。

### 2.5 時系列キャッシュ

- 使用されないプローブは **N フレーム保持**（`r.Lumen.RadianceCache.NumFramesToKeepCachedProbes = 8`）
- プローブの ProbeLastUsedFrame を毎フレーム更新、超過プローブはフリーリストに戻す

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `LumenRadianceCache.cpp` | 全体オーケストレーション、Clipmap 管理 |
| `LumenRadianceCache.h` | API（`GetInterpolationParameters` 等） |
| `LumenRadianceCacheInternal.h` | 内部構造体 |
| `LumenRadianceCacheHardwareRayTracing.cpp` | HW RT トレースパス |
| `LumenIrradianceFieldGather.cpp` | プローブから Irradiance アトラス計算 |
| `LumenVisualizeRadianceCache.cpp` | デバッグ可視化 |
| `LumenRadianceCacheCommon.ush` (shader) | 共有定数・マクロ |

### 3.2 主要定数（`LumenRadianceCache.cpp:109-113`）

```cpp
namespace LumenRadianceCache
{
    // Must match LumenRadianceCacheCommon.ush
    constexpr uint32 PRIORITY_HISTOGRAM_SIZE = 16;
    constexpr uint32 PROBES_TO_UPDATE_TRACE_COST_STRIDE = 2;
    ...
}
```

### 3.3 Interpolation パラメータ構築（`LumenRadianceCache.cpp:163-180`）

ScreenProbeGather 等の呼び出し側で:

```cpp
LumenRadianceCache::GetInterpolationParameters(
    View, GraphBuilder, RadianceCacheState, RadianceCacheInputs, OutParameters);
```

を呼び、shader バインディング `RadianceProbeIndirectionTexture` / `RadianceCacheFinalRadianceAtlas` 等を取得。レイの Trace 関数内で:

```hlsl
// 擬似コード
if (rayDistance > radianceCacheTMin)
{
    float3 worldPos = rayOrigin + rayDir * rayDistance;
    float3 radiance = SampleRadianceCache(worldPos, rayDir, radianceCacheParameters);
    return radiance;  // トレース打ち切り
}
```

### 3.4 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.Lumen.RadianceCache.Update` | 1 | 0 で更新停止（デバッグ） |
| `r.Lumen.RadianceCache.NumFramesToKeepCachedProbes` | 8 | 未使用プローブ保持フレーム数 |
| `r.Lumen.RadianceCache.SpatialFilterProbes` | 1 | 隣接プローブ平均 ON/OFF |
| `r.Lumen.RadianceCache.SpatialFilterMaxRadianceHitAngle` | 0.2° | フィルタ角度制限 |
| `r.Lumen.RadianceCache.SortTraceTiles` | 0 | 方向別ソートで coherency 改善 |
| `r.Lumen.RadianceCache.SupersampleTileBRDFThreshold` | 0.1 | supersample 切替閾値 |
| `r.Lumen.RadianceCache.SupersampleDistanceFromCamera` | -1 | 近距離のみ supersample |
| `r.Lumen.RadianceCache.DownsampleDistanceFromCamera` | 4000 cm | 遠方は強制 downsample |
| `r.Lumen.RadianceCache.ForceUniformTraceTileLevel` | -1 | デバッグ用、0=半 / 1=フル / 2=super |

### 3.5 Probe Free List

`LumenRadianceCache.cpp:262` `FClearProbeFreeList` shader が起動時にフリーリストを初期化:

```hlsl
// 全プローブインデックスをフリーリストに push、ProbeWorldOffset を 0 で初期化
```

毎フレーム `FUpdateCacheForUsedProbesCS` が:
- マークされたプローブを物理スロットに割当
- 未使用プローブを `NumFramesToKeepCachedProbes` 経過後にフリーに戻す

---

## 4. 近似・省略の差分

| 項目 | 理想 | UE 実装 | 影響 |
|------|------|--------|------|
| プローブ密度 | 連続 | Clipmap 4 段の Sparse 3D | 遠方で粗い、低周波なので問題小 |
| 球面表現 | 連続 | オクタヘドラル 32×32 | 高周波スペキュラ的な反射不可（diffuse GI 用途のみ） |
| 更新タイミング | 毎フレーム全更新 | priority + N フレーム保持 | 急激な光源変化で 1〜数フレーム遅延 |
| 遮蔽 | 完全 | プローブ単位の visibility なし | leak 発生、Probe Occlusion Atlas で軽減 |
| Trace 距離 | 無限 | プローブ TMin 越えで打ち切り | TMin 内は ScreenProbe / SDF にフォールバック |
| Spatial Filter | なし | 隣接平均（angle 制限） | leak 軽減 vs detail 損失のトレードオフ |

---

## 5. 代替手法との比較

| 手法 | プローブ配置 | 球面表現 | 動的更新 | 採用 |
|------|-----------|---------|--------|------|
| Static Light Probe (UE4) | エディタ手動配置 | SH3 | 不可 | UE5 でも一部互換 |
| Volumetric Lightmap | 3D グリッド ベイク | SH | 不可 | Static GI |
| DDGI / RTXGI | 3D グリッド | オクタヘドラル | フル更新 | NVIDIA RTXGI |
| **Lumen Radiance Cache** | **Clipmap Sparse 3D** | **オクタヘドラル + 重要度サンプリング** | **priority 駆動** | **UE5 標準** |
| Lumen Translucency Volume | フローズン 3D | SH | フル更新 | 半透明用 |

### DDGI vs Lumen Radiance Cache

- **DDGI**: 全プローブ毎フレーム同等更新 → 単純だが GPU 時間がプローブ数に比例
- **Lumen RC**: 「画面から実際に参照されたプローブだけ更新」 → スパース、可変予算

---

## 6. パラメータと CVar

§3.4 にまとめ済み。

---

## 7. 参考資料

- [S10] Wright / Heitz / Hillaire 2022 §5 — Radiance Cache 設計の原典
- 関連: Crytek "Sparse Voxel GI" (2014) — ボクセル系の系譜
- 関連: NVIDIA "DDGI" / "RTXGI" — グリッドプローブ系の比較対象

---

## 8. 相談用フック

- **理解度チェック**:
  - Surface Cache と Radiance Cache の役割分担 → Surface = 表面密着 2D、Radiance = 空間 3D プローブ
  - なぜ Clipmap 階層？ → カメラ近傍は高解像度、遠方は粗くて十分（視覚的重要度）
  - 「Trace Tile」とは → §2.4、プローブ方向を粒度分解した単位（half / full / super）
- **コード深掘り候補**:
  - `FUpdateCacheForUsedProbesCS` の Free List allocation ロジック
  - `LumenRadianceCacheHardwareRayTracing.cpp` でのプローブトレース pass
  - Probe Occlusion Atlas の生成と利用（leak 軽減のキー）
- **未読箇所**:
  - S10 §5.5 Probe Occlusion 計算式
  - `LumenRadianceCacheCommon.ush` の SampleRadianceCache 関数本体
  - Skylight との統合（`UseSkyVisibility` フラグ）
- **次の派生**:
  - Irradiance Field Gather → [[lumen_irradiance_field]]（未着手）
  - Translucency Volume Lighting → [[lumen_translucency_volume]]（未着手）
  - ScreenProbe からの参照経路 → [[lumen_final_gather]]
