# Substrate ソースマップ

- 対象: 新マテリアルレイヤーシステム（UE5.3〜 実験的、旧 Shading Models 置換）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[08_substrate_overview]]

1 ピクセルに複数のマテリアルクロージャを積み重ねる構造。BasePass で MaterialTextureArray（Texture2DArray<uint>）に
書き込み、ステンシルビットで Fast / Single / Complex タイル分類 → タイル単位 Indirect Dispatch でライティング。

---

## ソースパス

| 対象 | パス |
|------|------|
| フォルダ | `Engine/Source/Runtime/Renderer/Private/Substrate/` |
| メイン | `Substrate.h/.cpp` |
| 屈折 | `SubstrateRoughRefraction.cpp` |
| デバッグ | `SubstrateVisualize.cpp` |
| Glint | `Glint/`（マイクロファセットきらめき） |
| シェーダー | `Engine/Shaders/Private/Substrate/*.usf` |

---

## ファイル → クラス対応

### フレーム初期化

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `Substrate.h/.cpp` | `InitialiseSubstrateFrameSceneData()`, `FSubstrateSceneData`, `FSubstrateViewData`, `FSubstrateGlobalUniformParameters` | リソース確保・グローバル UBO バインド | [[a_substrate_material]] |
| `Substrate.h/.cpp` | `BindSubstrateBasePassUniformParameters()` | BasePass 用 UB 注入 | 同 |

### 分類（タイル）

| ファイル | 主要関数 / enum | 役割 | 参照 |
|---------|--------------|------|------|
| `Substrate.h/.cpp` | `AddSubstrateMaterialClassificationPass()`, `AddSubstrateMaterialClassificationIndirectArgsPass()`, `ESubstrateTileType`, `StencilBit_Fast/Single/Complex/ComplexSpecial` | ステンシルビット + タイルリスト + Indirect Args | [[b_substrate_classify]] |
| `Substrate.h/.cpp` | `FSubstrateTilePassVS`, `SetTileParameters()` | タイル展開 VS | 同 |

### ライティング関連

| ファイル | 主要関数 | 役割 | 参照 |
|---------|--------|------|------|
| `SubstrateRoughRefraction.cpp` | `AddSubstrateOpaqueRoughRefractionPasses()` | 不透明粗面屈折 | [[c_substrate_lighting]] |
| `SubstrateVisualize.cpp` | `AddSubstrateDebugPasses()` | デバッグ表示 | — |

---

## フレームライフサイクル

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─[A] InitialiseSubstrateFrameSceneData             Substrate.cpp
  │     ├─ EffectiveMaxBytesPerPixel / ClosurePerPixel 算出
  │     ├─ MaterialTextureArray（Texture2DArray<uint>）確保
  │     ├─ TopLayerTexture / ClosureOffsetTexture 確保
  │     └─ FSubstrateGlobalUniformParameters バインド
  │
  ├─[B] BasePass                                      BasePassRendering.cpp
  │     BindSubstrateBasePassUniformParameters
  │     → 各マテリアルシェーダーが MaterialTextureArrayUAV にクロージャ書き込み
  │
  ├─[C] AddSubstrateMaterialClassificationPass        Substrate.cpp
  │     ステンシルビット（0x10/0x20/0x40/0x80）書き込み
  │     ClassificationTileListBuffer + IndirectArgsBuffer 生成
  │     （r.Substrate.AsyncClassification=1 で Shadow パスと並行）
  │
  ├─[C'] AddSubstrateMaterialClassificationIndirectArgsPass
  │
  ├─[D] DeferredLighting                              Indirect Dispatch × 3
  │     ├─ SetTileParameters(Fast)    → Fast シェーダー（単純）
  │     ├─ SetTileParameters(Single)  → Single シェーダー
  │     └─ SetTileParameters(Complex) → Complex シェーダー（複数クロージャ）
  │
  ├─[E] AddSubstrateOpaqueRoughRefractionPasses       SubstrateRoughRefraction.cpp
  │     OpaqueRoughRefractionTexture → SceneColor 合成
  │
  └─[F] AddSubstrateDebugPasses                       SubstrateVisualize.cpp（デバッグ時）
```

---

## タイル分類（ステンシルビット）

| ビット | ESubstrateTileType | 用途 |
|--------|------------------|------|
| `0x10` | `Fast` | 単純マテリアル（最速） |
| `0x20` | `Single` | 中程度 |
| `0x40` | `Complex` | 複数クロージャ |
| `0x80` | `ComplexSpecial` | 特殊複雑（Eye 等） |

---

## 主要テクスチャ

| テクスチャ | フォーマット | 内容 |
|----------|------------|------|
| `MaterialTextureArray` | Texture2DArray<uint> | クロージャデータ（可変スライス数） |
| `TopLayerTexture` | 2D | 最上位レイヤー情報 |
| `OpaqueRoughRefractionTexture` | 2D | 粗面屈折 |
| `ClosureOffsetTexture` | 2D | クロージャオフセット |
| `SampledMaterialTexture` | 2D | サンプル済みマテリアル |

---

## 主要 CVar

| CVar | 対応ファイル |
|------|------------|
| `r.Substrate` | （プロジェクト設定） |
| `r.Substrate.BytesPerPixel` | `Substrate.cpp` |
| `r.Substrate.ClosurePerPixel` | 同 |
| `r.Substrate.AllocationMode` | 同 |
| `r.Substrate.AsyncClassification` | 同 |
| `r.Substrate.Debug.AdvancedVisualization` | `SubstrateVisualize.cpp` |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Details | [[a_substrate_material]] | クロージャ / MaterialTextureArray 書き込み |
| Details | [[b_substrate_classify]] | タイル分類パス |
| Details | [[c_substrate_lighting]] | ライティング / RoughRefraction / SubSurface |

---

## ue5-dive 起点

- 「Substrate のエントリ」 → `Substrate.cpp:InitialiseSubstrateFrameSceneData`
- 「BasePass への注入」 → `BindSubstrateBasePassUniformParameters`
- 「タイル分類」 → `AddSubstrateMaterialClassificationPass` + ステンシルビット
- 「タイル展開 VS」 → `FSubstrateTilePassVS` + `SetTileParameters(ESubstrateTileType)`
- 「クロージャ上限」 → `r.Substrate.BytesPerPixel` / `r.Substrate.ClosurePerPixel`
- 「旧 Shading Models との関係」 → 有効時は Substrate がライティングパスを置き換え
