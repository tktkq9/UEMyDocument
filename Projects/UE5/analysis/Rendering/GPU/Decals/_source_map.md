# GPU Decals ソースマップ

- 対象: Deferred Decals GPU シェーダー（DBuffer + GBuffer 後置）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_decals_gpu_overview]]

DBuffer は BasePass 前に Pre-GBuffer 合成。GBuffer 後置は BlendMode 別（Translucent/Stain/Normal/Emissive）
ステージで実行。どちらも `DeferredDecal.usf` の MainVS/MainPS を共用しパーミュテーションで分岐。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/DeferredDecal.usf` |
| CPU 呼び出し | `Renderer/Private/DecalRenderingShared.cpp` / `DeferredDecal.cpp` |

---

## ファイル → シェーダー対応

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `DeferredDecal.usf` | `MainVS()` | `RenderDeferredDecals()` | デカールボックス投影 → クリップ座標 | [[detail_deferred_decal]], [[detail_dbuffer]] |
| `DeferredDecal.usf` | `FPixelShaderInOut_MainPS()` | 同上 | SceneDepth → ワールド座標 → デカール空間 → DBuffer or GBuffer 書き込み | 同 |

---

## GPU データフロー

```
[1] DBuffer パス（BasePass 前）
    EDecalRenderStage::BeforeBasePass
    DeferredDecal.usf:MainVS → FPixelShaderInOut_MainPS
      → DBufferA（BaseColor）/ DBufferB（Normal）/ DBufferC（Roughness/Metallic）

[2] Deferred Decal パス（GBuffer 後置）
    EDecalRenderStage::BeforeLighting / AfterBasePass / Emissive / AO 等
    DeferredDecal.usf:MainVS → FPixelShaderInOut_MainPS（BlendMode 別パーミュテーション）
      → GBuffer（BaseColor/Normal/Roughness/Metallic）または SceneColor（Emissive/Translucent）
```

---

## 主要パーミュテーション

| マクロ | 用途 |
|--------|------|
| `DECAL_BLEND_MODE=0` | Translucent |
| `DECAL_BLEND_MODE=1` | Stain |
| `DECAL_BLEND_MODE=2` | Normal |
| `DECAL_BLEND_MODE=3` | Emissive |
| `DECAL_RENDERTARGET_COUNT_*` | 書き込み RT 数 |
| `DECAL_RENDERSTAGE_*` | ステージ別分岐 |

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_deferred_decal]] / [[detail_dbuffer]] |
| Reference | [[ref_deferred_decal]] / [[ref_dbuffer]] |

---

## ue5-dive 起点

- 「デカール VS」 → `DeferredDecal.usf:MainVS`
- 「デカール PS」 → `DeferredDecal.usf:FPixelShaderInOut_MainPS`
- 「DBuffer ステージ分岐」 → `EDecalRenderStage::BeforeBasePass` + `DECAL_RENDERSTAGE_*`
- 「BlendMode 分岐」 → `DECAL_BLEND_MODE` パーミュテーション
