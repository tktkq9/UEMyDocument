# GPU DepthPrepass / Velocity ソースマップ

- 対象: Depth Pre-pass + Velocity Buffer GPU シェーダー
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]] / [[01_gpu_overview]]
- 概要: [[01_depthprepass_gpu_overview]]

Opaque は位置ストリームのみで深度書き込み。Masked はフルストリーム + Opacity テスト。
Velocity は現/前フレーム位置差分から R16G16F に書き出し。Nanite は CS で VisBuffer から復元。

---

## ソースパス

| 対象 | パス |
|------|------|
| シェーダー | `Engine/Shaders/Private/DepthOnlyVertexShader.usf` |
| シェーダー | `Engine/Shaders/Private/DepthOnlyPixelShader.usf` |
| シェーダー | `Engine/Shaders/Private/VelocityShader.usf` |
| CPU | `Renderer/Private/DepthRendering.cpp` / `VelocityRendering.cpp` |

---

## ファイル → シェーダー対応

### Depth Only

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `DepthOnlyVertexShader.usf` | `Main()` | `FDeferredShadingSceneRenderer::RenderPrePassView()` | 位置ストリーム → 深度のみ | [[detail_depth_only]] |
| `DepthOnlyPixelShader.usf` | `Main()` | 同上（Masked 時のみ） | Opacity マスク + PDO | 同 |

### Velocity

| シェーダーファイル | 主要エントリ | CPU 関数 | 役割 | 参照 |
|----------------|-----------|---------|------|------|
| `VelocityShader.usf` | `MainVertexShader()` | `FDeferredShadingSceneRenderer::RenderVelocities()` | 現/前フレーム座標補間 | [[detail_velocity]] |
| `VelocityShader.usf` | `MainPixelShader()` | 同上 | Velocity = (CurPos - PrevPos)/2 書き出し | 同 |

---

## GPU データフロー

```
[1] Depth Pre-pass
    [Opaque]  DepthOnlyVertexShader.usf:Main → 深度のみ（PS 無し）
    [Masked]  DepthOnlyVertexShader.usf:Main + DepthOnlyPixelShader.usf:Main
                 → Opacity 評価 + PDO 適用
    → SceneDepthBuffer

[2] Velocity Pass
    VelocityShader.usf:MainVertexShader → MainPixelShader
    → VelocityBuffer (R16G16F)

[3] Nanite Velocity
    CS ベース（NaniteExport 系）で VisBuffer から復元
```

---

## 主要パーミュテーション

| マクロ | 用途 |
|--------|------|
| `MATERIALBLENDING_MASKED` | Masked 時の PS 有効 |
| `USE_PDO` | Pixel Depth Offset |
| `VELOCITY_USES_PREV_XFORM` | スケルタル速度 |

---

## Details/Reference 一覧

| 種別 | ファイル |
|------|---------|
| Details | [[detail_depth_only]] / [[detail_velocity]] |
| Reference | [[ref_depth_only]] / [[ref_velocity]] |

---

## ue5-dive 起点

- 「Depth Only VS」 → `DepthOnlyVertexShader.usf:Main`
- 「Masked 深度 PS」 → `DepthOnlyPixelShader.usf:Main` + `MATERIALBLENDING_MASKED`
- 「Velocity 書き出し」 → `VelocityShader.usf:MainPixelShader`
