# DistanceField（SDF）全体概要

- 取得日: 2026-04-18
- 対象: `Engine/Source/Runtime/Engine/Public/DistanceFieldAtlas.h`, `Engine/Source/Runtime/Renderer/Private/GlobalDistanceField.*`, `DistanceFieldAmbientOcclusion.*`, `DistanceFieldShadowing.*`
- 上位: [[01_rendering_overview]]

---

## SDF とは

**Signed Distance Field（SDF）** は各ボクセルに「最近傍サーフェスまでの符号付き距離」を格納したボリュームテクスチャ。  
UE5 では Lumen 専用ではなく、複数のレンダリングシステムが共通基盤として利用する。

| 利用システム | 使用する SDF | 主な用途 |
|------------|------------|---------|
| **Mesh SDF** | メッシュごとのオフライン生成 SDF | DFAO / DFShadows / Lumen のトレース基盤 |
| **Global SDF** | ランタイム合成のワールド SDF（クリップマップ） | DFAO コーントレース・Lumen SurfaceCache |
| **DFAO** | Global SDF | Sky Light 遮蔽の AO 計算 |
| **DF Shadows** | Mesh SDF | ソフトシャドウ（コーントレース） |
| **Lumen** | Mesh SDF + Global SDF | GI トレースバックエンド |

---

## データフロー

```
【オフライン（ビルド時）】
UStaticMesh
  └─ FDistanceFieldAsyncQueue::Build()
       └─ GenerateSignedDistanceFieldVolumeData()
            → FDistanceFieldVolumeData（Mips + AlwaysLoadedMip）
               ├─ FDistanceFieldSceneData::Atlas へパック
               └─ GPU へアップロード（UpdateDistanceFieldAtlas）

【ランタイム（毎フレーム）】
FDistanceFieldSceneData::UpdateDistanceFieldObjectBuffers()
  → Object SDF バッファを GPU に転送

UpdateGlobalDistanceFieldVolume()
  → Mesh SDF を合成してクリップマップ（4段）に書き込み
     ├─ GDF_MostlyStatic  : 静的オブジェクト専用クリップマップ
     └─ GDF_Full          : 動的オブジェクトも含む完全クリップマップ

【利用（描画パス）】
DFAO ────── Global SDF コーントレース → AO テクスチャ
DFShadows ── Mesh SDF コーントレース  → ソフトシャドウ
Lumen ─────── Mesh SDF + Global SDF   → GI トレース
```

---

## クリップマップ構成

```
Global SDF クリップマップ（Lumen 有効時：4〜8 段、無効時：4 段）

  Clipmap[0]  近距離（高解像度・狭範囲）
  Clipmap[1]  中距離
  Clipmap[2]  遠距離
  Clipmap[3]  最遠（低解像度・広範囲）
  ...

  カメラ移動に追従してスライディング更新
  大きな移動やカメラカット時は全面更新
```

---

## CPU / GPU の役割分担

| 処理 | CPU 側 | GPU 側 |
|------|--------|--------|
| Mesh SDF ビルド | `FDistanceFieldAsyncQueue`（非同期） | — |
| Atlas テクスチャ更新 | `UpdateDistanceFieldAtlas()` | Compute（ScatterUpload） |
| Object バッファ更新 | `UpdateDistanceFieldObjectBuffers()` | Compute（ScatterUpload） |
| Global SDF 更新 | `UpdateGlobalDistanceFieldVolume()` | Compute（`GlobalDistanceField.usf`） |
| DFAO レンダリング | `RenderDistanceFieldLighting()` | Compute（`DistanceFieldShadowing.usf`） |
| DF Shadows | `RenderRayTracedDistanceFieldProjection()` | Compute（`DistanceFieldShadowing.usf`） |

---

## 主要ソースファイル

| ファイル | 内容 |
|---------|------|
| `Engine/Public/DistanceFieldAtlas.h` | `FDistanceFieldVolumeData`, `FDistanceFieldAsyncQueue` |
| `Renderer/Private/ScenePrivate.h:2070` | `FDistanceFieldSceneData` |
| `Renderer/Private/GlobalDistanceField.h/.cpp` | Global SDF 更新・クリップマップ管理 |
| `Renderer/Private/DistanceFieldAmbientOcclusion.h/.cpp` | DFAO パス |
| `Renderer/Private/DistanceFieldShadowing.cpp` | DF Shadows パス |
| `Renderer/Private/DistanceFieldLightingShared.h` | 共通パラメータ構造体 |

---

## 関連ドキュメント

| ドキュメント | 内容 |
|------------|------|
| [[a_mesh_sdf]] | Mesh SDF 構造・オフラインビルド・Atlas 管理 |
| [[b_global_sdf]] | Global SDF クリップマップ・UpdateGlobalDistanceFieldVolume |
| [[c_dfao]] | Distance Field AO パス詳細 |
| [[d_df_shadows]] | Distance Field Shadows パス詳細 |
| [[ref_df_atlas]] | FDistanceFieldVolumeData / FDistanceFieldSceneData リファレンス |
| [[ref_global_sdf]] | FGlobalDistanceFieldInfo / GlobalDistanceField namespace リファレンス |
| [[ref_dfao]] | DFAO エントリポイントリファレンス |
| [[ref_df_shadows]] | DF Shadows エントリポイントリファレンス |
