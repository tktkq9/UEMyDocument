# GPU BasePass ソースマップ

- 対象: BasePass GPU シェーダー（GBuffer 書き込み + Nanite + Substrate）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_basepass_gpu_overview]]

通常メッシュは BasePassVS/PS で GBuffer 直書き。Nanite は VisBuffer → Shade Binning → Export。
Substrate は BasePass PS 内マクロで MaterialTextureArray にクロージャをエンコード。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/BasePassVertexShader.usf` |
| シェーダー | `Engine/Shaders/Private/BasePassPixelShader.usf` |
| Nanite | `Engine/Shaders/Private/Nanite/NaniteShadeBinning.usf` |
| Nanite | `Engine/Shaders/Private/Nanite/NaniteExportGBuffer.usf` |
| Substrate | `Engine/Shaders/Private/Substrate/Substrate.ush` |
| CPU 呼び出し | `Renderer/Private/BasePassRendering.cpp` |

---

## ファイル → シェーダー対応

### 通常メッシュ BasePass

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `BasePassVertexShader.usf` | `Main()` | `FDeferredShadingSceneRenderer::RenderBasePass()` | WPO・接線空間・ライトマップ UV | [[detail_opaque]], [[ref_opaque]] |
| `BasePassPixelShader.usf` | `FPixelShaderInOut_MainPS()` | 同上 | マテリアル評価 → GBufferA/B/C/D/E/F/Velocity | 同 |

### Nanite マテリアル GBuffer

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `Nanite/NaniteShadeBinning.usf` | `ShadeBinBuildCS()` / `ShadeBinReserveCS()` | `Nanite::DispatchShading()` | VisBuffer ピクセルをシェーディングビン別に分類 | [[detail_nanite_material]], [[ref_nanite_material]] |
| `Nanite/NaniteExportGBuffer.usf` | `EmitSceneDepthPS()` / `EmitSceneStencilPS()` | `Nanite::DrawBasePass()` | Nanite VisBuffer → SceneDepth/Stencil 変換 | 同 |

### Substrate GBuffer エンコード

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `Substrate/Substrate.ush` | BasePass PS から内部呼び出し（`SUBSTRATE_ENABLED=1`） | `RenderBasePass()` | マテリアルレイヤーを SLAB 構造に変換・MaterialTextureArray 書き込み | [[detail_substrate]], [[ref_substrate]] |

---

## GPU データフロー

```
[1] 通常メッシュ
    BasePassVertexShader.usf:Main()
      → WPO / 接線空間 / ライトマップ UV
    BasePassPixelShader.usf:FPixelShaderInOut_MainPS()
      → マテリアル評価 → GBufferA/B/C/D/E/F + Velocity

[2] Nanite
    NaniteShadeBinning.usf:ShadeBinBuildCS → ShadeBinReserveCS
      → VisBuffer ピクセル分類
    NaniteExportGBuffer.usf:EmitSceneDepthPS / EmitSceneStencilPS
      → SceneDepth / Stencil への変換

[3] Substrate
    BasePassPixelShader.usf から Substrate.ush マクロ呼び出し
      → MaterialTextureArray<uint> へクロージャエンコード
```

---

## 主要パーミュテーション

| マクロ | 用途 |
|--------|------|
| `MATERIAL_LIT` | ライティング有り |
| `MATERIAL_UNLIT` | アンリット |
| `IS_NANITE_PASS` | Nanite 経路 |
| `SUBSTRATE_ENABLED` | Substrate 有効 |
| `SIMPLIFIED_MATERIAL_SHADER` | 簡易 |

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_opaque]] / [[detail_nanite_material]] / [[detail_substrate]] |
| Reference | [[ref_opaque]] / [[ref_nanite_material]] / [[ref_substrate]] |

---

## ue5-dive 起点

- 「BasePass VS の WPO」 → `BasePassVertexShader.usf:Main`
- 「GBuffer 書き込み」 → `BasePassPixelShader.usf:FPixelShaderInOut_MainPS`
- 「Nanite の GBuffer 経路」 → `NaniteShadeBinning.usf:ShadeBinBuildCS` → `NaniteExportGBuffer.usf`
- 「Substrate クロージャ」 → `Substrate.ush` + `SUBSTRATE_ENABLED`
