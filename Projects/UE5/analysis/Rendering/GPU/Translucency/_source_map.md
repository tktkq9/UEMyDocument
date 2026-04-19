# GPU Translucency ソースマップ

- 対象: Translucency GPU シェーダー（Forward Shading + Lighting Volume + OIT）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_translucency_gpu_overview]]

半透明フォワードシェーディング・ライティングボリューム（4段カスケード 3D テクスチャ）注入・
OIT（深度ソートなし半透明）の 3 系統。OIT はサンプルソート + 前後ブレンドで順序独立を実現。

---

## ソースパス

| 対象 | パス |
|------|------|
| Forward | `Engine/Shaders/Private/BasePassPixelShaders.usf`（Translucent パーミュテーション） |
| Compose | `Engine/Shaders/Private/ComposeSeparateTranslucency.usf` |
| Lighting | `Engine/Shaders/Private/TranslucentLightInjectionShaders.usf` |
| Lighting | `Engine/Shaders/Private/TranslucentLightingShaders.usf` |
| OIT | `Engine/Shaders/Private/OIT/OITSorting.usf` |
| OIT | `Engine/Shaders/Private/OIT/OITCombine.usf` |
| CPU | `Renderer/Private/TranslucentRendering.cpp` / `TranslucentLighting.cpp` / `OIT/*` |

---

## ファイル → シェーダー対応

### TranslucentForward

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `BasePassPixelShaders.usf` | `BasePassPS`（Translucent パーミュテーション） | `RenderTranslucency()` | 深度書き込み無し・アルファブレンド・SeparateTranslucency 半解像度書き込み | [[detail_translucent_forward]] |
| `ComposeSeparateTranslucency.usf` | `MainPS()` | 同上 | SceneColor に半透明結果を合成 | 同 |

### TranslucentLighting

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `TranslucentLightInjectionShaders.usf` | `InjectMainPS` / `InjectBatchMainCS` | `RenderTranslucencyLighting()` | ライト輝度を 4 段カスケード 3D ボリュームに注入 | [[detail_translucent_lighting]] |
| `TranslucentLightingShaders.usf` | `FilterTranslucentVolumeCS` | 同上 | 隣接ボクセル間ガウスフィルタ | 同 |

### OIT

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `OIT/OITSorting.usf` | `MainCS`（複数パーミュテーション） | `RenderOIT()` | サンプルを深度でソート | [[detail_oit]] |
| `OIT/OITCombine.usf` | `MainCS` | 同上 | ソート済みサンプルを前後ブレンド合成 | 同 |

---

## GPU データフロー

```
[TranslucentForward]                RenderTranslucency
  BasePassPixelShaders.usf（Translucent）
    → SeparateTranslucency テクスチャ書き込み（半解像度）
  ComposeSeparateTranslucency.usf:MainPS
    → SceneColor 合成

[TranslucentLighting]               RenderTranslucencyLighting
  TranslucentLightInjectionShaders.usf:InjectMainPS / InjectBatchMainCS
    → TranslucencyLightingVolume（4 段カスケード 3D）
  TranslucentLightingShaders.usf:FilterTranslucentVolumeCS
    → 空間ガウスフィルタ
  （半透明マテリアルが Lookup してライティングを適用）

[OIT]                               RenderOIT
  OIT/OITSorting.usf:MainCS
    → 深度ソート
  OIT/OITCombine.usf:MainCS
    → 前後ブレンド合成
```

---

## 主要パーミュテーション

| マクロ | 用途 |
|--------|------|
| `MATERIALBLENDING_TRANSLUCENT` | 半透明ブレンド |
| `SEPARATE_TRANSLUCENCY` | 半解像度分離半透明 |
| `TRANSLUCENCY_LIGHTING_VOLUMETRIC_NONDIRECTIONAL` | 非方向ボリューム |
| `TRANSLUCENCY_LIGHTING_VOLUMETRIC_DIRECTIONAL` | 方向ボリューム |
| `OIT_SORTING_MODE` | OIT ソートモード |

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_translucent_forward]] / [[detail_translucent_lighting]] / [[detail_oit]] |
| Reference | [[ref_translucent_forward]] / [[ref_translucent_lighting]] / [[ref_oit]] |

---

## ue5-dive 起点

- 「半透明 BasePass」 → `BasePassPixelShaders.usf` + Translucent パーミュテーション
- 「SeparateTranslucency 合成」 → `ComposeSeparateTranslucency.usf:MainPS`
- 「ライトボリューム注入」 → `TranslucentLightInjectionShaders.usf:InjectMainPS`
- 「ボリュームフィルタ」 → `TranslucentLightingShaders.usf:FilterTranslucentVolumeCS`
- 「OIT ソート」 → `OIT/OITSorting.usf:MainCS`
