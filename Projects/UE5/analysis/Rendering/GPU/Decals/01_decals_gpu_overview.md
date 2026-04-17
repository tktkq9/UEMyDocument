# Deferred Decals GPU 処理概要

- グループ: Decals GPU
- CPU 概要: [[15_decals_overview]]
- CPU 詳細: [[a_deferred_decal]] | [[b_dbuffer]]

---

## Deferred Decals GPU パイプライン実行順

UE5 の Deferred Decals は **DBuffer（Pre-GBuffer 合成）** と **GBuffer 後置デカール** の2系統がある。

```
[DBuffer デカール（Pre-GBuffer）]
[1]  DBuffer Pass
     ├─ MainVS（DeferredDecal.usf）… デカールボックスを投影してクリップ座標に変換
     └─ FPixelShaderInOut_MainPS（DeferredDecal.usf）
          SceneDepth からワールド座標を復元 → デカール空間に変換 → DBufferA/B/C に書き込み

[GBuffer 後置デカール]
[2]  Deferred Decal Pass（BlendMode 別）
     ├─ Translucent / Stain / Normal / Emissive の各ステージで実行
     └─ MainVS + FPixelShaderInOut_MainPS（DeferredDecal.usf）
          GBuffer を読み取り → デカールマテリアルで上書き
```

---

## 各ステップの詳細

### [1] DBuffer Pass

| 項目 | 内容 |
|-----|------|
| **概要** | BasePass の前に実行。デカールマテリアルの BaseColor/Normal/Roughness を DBuffer に積算 |
| **CPU 関数** | `RenderDeferredDecals()` with `EDecalRenderStage::BeforeBasePass` |
| **シェーダー** | VS: `DeferredDecal.usf#MainVS()`; PS: `DeferredDecal.usf#FPixelShaderInOut_MainPS()` |
| **出力** | DBufferA（BaseColor）/ DBufferB（Normal）/ DBufferC（Roughness/Metallic）|
| **CPU 詳細** | [[b_dbuffer]] |
| **GPU シェーダー詳細** | [[detail_dbuffer]] / [[ref_dbuffer]] |

---

### [2] Deferred Decal Pass

| 項目 | 内容 |
|-----|------|
| **概要** | GBuffer 後にデカールを投影合成。ライティング前後・Emissive・AO ステージ別に実行 |
| **CPU 関数** | `RenderDeferredDecals()` with `EDecalRenderStage::BeforeLighting` 等 |
| **シェーダー** | VS: `DeferredDecal.usf#MainVS()`; PS: `DeferredDecal.usf#FPixelShaderInOut_MainPS()` |
| **出力** | GBuffer（BaseColor/Normal/Roughness/Metallic）or SceneColor（Emissive/Translucent）|
| **CPU 詳細** | [[a_deferred_decal]] |
| **GPU シェーダー詳細** | [[detail_deferred_decal]] / [[ref_deferred_decal]] |

---

## シェーダー別 CPU 対応一覧

| シェーダーファイル | 主要エントリポイント | CPU 関数 | グループ |
|----------------|----------------|---------|---------|
| `DeferredDecal.usf` | `MainVS()` | `RenderDeferredDecals()` | [[a_DeferredDecal]] / [[b_DBuffer]] |
| `DeferredDecal.usf` | `FPixelShaderInOut_MainPS()` | `RenderDeferredDecals()` | [[a_DeferredDecal]] / [[b_DBuffer]] |
