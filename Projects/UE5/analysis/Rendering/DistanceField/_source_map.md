# DistanceField ソースマップ

- 対象: Mesh SDF / Global SDF（クリップマップ）/ DFAO / DF Shadows
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[10_distance_field_overview]]

Mesh SDF は StaticMesh のオフラインビルドで Atlas にパックされ、Global SDF は毎フレーム 4〜8 段のクリップマップに合成される。
DFAO / DF Shadows / Lumen の共通トレース基盤。

---

## ソースパス

| 対象 | パス |
|------|------|
| Atlas（Engine） | `Engine/Source/Runtime/Engine/Public/DistanceFieldAtlas.h` |
| Scene データ | `Engine/Source/Runtime/Renderer/Private/ScenePrivate.h`（`FDistanceFieldSceneData`） |
| Global SDF | `Engine/Source/Runtime/Renderer/Private/GlobalDistanceField.h/.cpp` |
| DFAO | `Engine/Source/Runtime/Renderer/Private/DistanceFieldAmbientOcclusion.h/.cpp` |
| DF Shadows | `Engine/Source/Runtime/Renderer/Private/DistanceFieldShadowing.cpp` |
| 共通 | `Engine/Source/Runtime/Renderer/Private/DistanceFieldLightingShared.h` |
| シェーダー | `Engine/Shaders/Private/DistanceField/*.usf`, `GlobalDistanceField.usf`, `DistanceFieldShadowing.usf` |

---

## ファイル → クラス対応

### Mesh SDF / Atlas

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `DistanceFieldAtlas.h` | `FDistanceFieldVolumeData`, `FDistanceFieldAsyncQueue` | メッシュ単位 SDF・非同期ビルドキュー | [[Reference/ref_df_atlas]] |
| `ScenePrivate.h` | `FDistanceFieldSceneData` (line 2070) | GPU Object バッファ / Atlas | 同 |
| — | `UpdateDistanceFieldAtlas()`, `UpdateDistanceFieldObjectBuffers()` | GPU アップロード（ScatterUpload） | 同 |

### Global SDF（クリップマップ）

| ファイル | 主要関数 / 構造体 | 役割 | 参照 |
|---------|----------------|------|------|
| `GlobalDistanceField.h/.cpp` | `UpdateGlobalDistanceFieldVolume()`, `FGlobalDistanceFieldInfo`, `GlobalDistanceField` namespace | クリップマップ更新・スライディング | [[Reference/ref_global_sdf]], [[b_global_sdf]] |

### DFAO

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `DistanceFieldAmbientOcclusion.h/.cpp` | `RenderDistanceFieldLighting()` | Global SDF コーントレース AO | [[Reference/ref_dfao]], [[c_dfao]] |

### DF Shadows

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `DistanceFieldShadowing.cpp` | `RenderRayTracedDistanceFieldProjection()` | Mesh SDF コーントレースソフトシャドウ | [[Reference/ref_df_shadows]], [[d_df_shadows]] |

---

## データフロー

```
【オフライン（ビルド時）】
UStaticMesh
  └─ FDistanceFieldAsyncQueue::Build()
       └─ GenerateSignedDistanceFieldVolumeData()
            → FDistanceFieldVolumeData（Mips + AlwaysLoadedMip）
               → FDistanceFieldSceneData::Atlas にパック → GPU Upload

【ランタイム（毎フレーム）】
FDistanceFieldSceneData::UpdateDistanceFieldObjectBuffers()
  → Object SDF バッファ転送

UpdateGlobalDistanceFieldVolume()                    GlobalDistanceField.cpp
  → Mesh SDF を合成してクリップマップ 4〜8 段に書き込み
     ├─ GDF_MostlyStatic  : 静的オブジェクト専用
     └─ GDF_Full          : 動的含む完全クリップマップ

【利用（描画パス）】
DFAO       → RenderDistanceFieldLighting          Global SDF コーントレース
DFShadows  → RenderRayTracedDistanceFieldProjection Mesh SDF コーントレース
Lumen      → LumenMeshSDF.cpp / LumenGlobalDistanceField.cpp 両方を参照
```

---

## クリップマップ構成

```
Clipmap[0]  近距離（高解像度・狭範囲）
Clipmap[1]  中距離
Clipmap[2]  遠距離
Clipmap[3]  最遠（低解像度・広範囲）
  ... Lumen 有効時は最大 8 段
  カメラ移動に追従してスライディング更新
  大きな移動 / カット時は全面更新
```

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.DistanceFieldAO` | `DistanceFieldAmbientOcclusion.cpp` |
| `r.DistanceFieldShadowing` | `DistanceFieldShadowing.cpp` |
| `r.GenerateMeshDistanceFields` | `DistanceFieldAtlas.cpp` |
| `r.DistanceFields.MaxAtlasDepth` | 同 |
| `r.GlobalDistanceField.ClipmapResolution` | `GlobalDistanceField.cpp` |
| `r.AOGlobalDistanceField` | 同 |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_df_atlas]] | FDistanceFieldVolumeData / FDistanceFieldSceneData |
| Reference | [[Reference/ref_global_sdf]] | FGlobalDistanceFieldInfo / GlobalDistanceField namespace |
| Reference | [[Reference/ref_dfao]] | RenderDistanceFieldLighting |
| Reference | [[Reference/ref_df_shadows]] | DF Shadows エントリ |
| Details | [[a_mesh_sdf]] | Mesh SDF 構造・Atlas 管理 |
| Details | [[b_global_sdf]] | クリップマップ更新 |
| Details | [[c_dfao]] | DFAO 詳細 |
| Details | [[d_df_shadows]] | DF Shadows 詳細 |

---

## ue5-dive 起点

- 「Mesh SDF のビルド」 → `FDistanceFieldAsyncQueue::Build` + `GenerateSignedDistanceFieldVolumeData`
- 「Global SDF のクリップマップ更新」 → `GlobalDistanceField.cpp:UpdateGlobalDistanceFieldVolume`
- 「DFAO の実体」 → `DistanceFieldAmbientOcclusion.cpp:RenderDistanceFieldLighting`
- 「DF Shadows の実体」 → `DistanceFieldShadowing.cpp:RenderRayTracedDistanceFieldProjection`
- 「Lumen との共有基盤」 → `LumenMeshSDF.cpp` / `LumenGlobalDistanceField.cpp` が同じデータを参照
- 「GPU Scene アップロード」 → `UpdateDistanceFieldObjectBuffers`（ScatterUpload）
