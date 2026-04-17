# Base Pass GPU 処理概要

- グループ: BasePass GPU
- CPU 概要: [[08_basepass_overview]]
- CPU 詳細: [[a_basepass_gbuffer]] | [[b_nanite_gbuffer]] | [[c_substrate_gbuffer]]

---

## Base Pass GPU パイプライン実行順

Base Pass は **GBuffer（ジオメトリバッファ）** に各メッシュのマテリアル属性を書き込む。  
Nanite メッシュは独自の ShadeBinning を経由して GBuffer を埋める。  
Substrate マテリアルは別のエンコーディングで GBuffer を書き込む。

```
[Opaque BasePass]
[1]  通常メッシュ GBuffer 書き込み
     ├─ Main（BasePassVertexShader.usf）… WPO・接線空間・ライトマップ UV を計算
     └─ FPixelShaderInOut_MainPS（BasePassPixelShader.usf）
          マテリアル評価 → GBufferA/B/C/D/E/F/Velocity に書き込み

[Nanite GBuffer]
[2]  Nanite Material GBuffer
     ├─ ShadeBinBuildCS / ShadeBinReserveCS（NaniteShadeBinning.usf）
     │   VisibilityBuffer のピクセルをシェーディングビン別に分類
     └─ EmitSceneDepthPS / EmitSceneStencilPS（NaniteExportGBuffer.usf）
          Nanite VisibilityBuffer → SceneDepth / Stencil に変換
     ※ 実際のマテリアルシェーディングは LumenCardComputeShader.usf 相当の
       Nanite Compute Shading（NaniteShading）で実行

[Substrate]
[3]  Substrate Material GBuffer エンコード
     └─ Substrate GBuffer Encoding（Substrate.ush 内のマクロで BasePassPixelShader から呼ばれる）
          マテリアルレイヤーを SLAB 構造に変換してエンコード
```

---

## 各ステップの詳細

### [1] Opaque BasePass

| 項目 | 内容 |
|-----|------|
| **概要** | 通常の不透明メッシュを GBuffer に書き込む。DBuffer デカールも適用 |
| **CPU 関数** | `FDeferredShadingSceneRenderer::RenderBasePass()` |
| **シェーダー** | VS: `BasePassVertexShader.usf#Main()`; PS: `BasePassPixelShader.usf` |
| **出力** | GBufferA（法線）/ GBufferB（メタリック等）/ GBufferC（BaseColor）/ SceneDepth |
| **CPU 詳細** | [[a_basepass_gbuffer]] |
| **GPU シェーダー詳細** | [[detail_opaque]] / [[ref_opaque]] |

---

### [2] Nanite Material GBuffer

| 項目 | 内容 |
|-----|------|
| **概要** | Nanite の VisibilityBuffer からシェーディングビン別にマテリアルを評価 |
| **CPU 関数** | `Nanite::DispatchShading()` |
| **シェーダー** | `NaniteShadeBinning.usf` / `NaniteExportGBuffer.usf` |
| **出力** | SceneDepth / GBuffer（VisibilityBuffer → マテリアル評価経由）|
| **CPU 詳細** | [[b_nanite_gbuffer]] |
| **GPU シェーダー詳細** | [[detail_nanite_material]] / [[ref_nanite_material]] |

---

### [3] Substrate GBuffer エンコード

| 項目 | 内容 |
|-----|------|
| **概要** | Substrate マテリアルレイヤーを SLAB 構造にエンコードして GBuffer に格納 |
| **CPU 関数** | BasePass PS から自動呼び出し（`SUBSTRATE_ENABLED=1` 時）|
| **シェーダー** | `Substrate/Substrate.ush` マクロ（BasePassPixelShader.usf に include される）|
| **CPU 詳細** | [[c_substrate_gbuffer]] |
| **GPU シェーダー詳細** | [[detail_substrate]] / [[ref_substrate]] |

---

## シェーダー別 CPU 対応一覧

| シェーダーファイル | 主要エントリポイント | CPU 関数 | グループ |
|----------------|----------------|---------|---------|
| `BasePassVertexShader.usf` | `Main()` | `RenderBasePass()` | [[a_Opaque]] |
| `BasePassPixelShader.usf` | `FPixelShaderInOut_MainPS()` | `RenderBasePass()` | [[a_Opaque]] |
| `Nanite/NaniteShadeBinning.usf` | `ShadeBinBuildCS()` | `Nanite::DispatchShading()` | [[b_NaniteMaterial]] |
| `Nanite/NaniteExportGBuffer.usf` | `EmitSceneDepthPS()` | `Nanite::DrawBasePass()` | [[b_NaniteMaterial]] |
| `Substrate/Substrate.ush` | BasePass PS から内部呼び出し | `RenderBasePass()` | [[c_Substrate]] |
