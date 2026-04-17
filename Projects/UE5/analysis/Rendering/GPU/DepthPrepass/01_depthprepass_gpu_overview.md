# Depth Pre-pass / Velocity Buffer GPU 処理概要

- グループ: DepthPrepass GPU
- CPU 概要: [[07_depthprepass_overview]]
- CPU 詳細: [[a_depth_prepass]] | [[b_velocity_buffer]]

---

## Depth Pre-pass GPU パイプライン実行順

Depth Pre-pass は **SceneDepth と Velocity Buffer** を BasePass より先に生成する。  
Opaque は位置のみのストリームを使い高速描画。Masked はフルストリームで Opacity テスト。

```
[Depth Pre-pass]
[1]  Depth Only Pass（不透明メッシュ）
     ├─ [Opaque]  Main（DepthOnlyVertexShader.usf）… 位置のみストリーム → 深度のみ書き込み
     └─ [Masked]  Main（DepthOnlyVertexShader.usf）+ Main（DepthOnlyPixelShader.usf）
                  … フルストリーム → Opacity マスク + PDO 対応

[Velocity Buffer]
[2]  Velocity Pass（動くメッシュ）
     ├─ MainVertexShader（VelocityShader.usf）… 現フレーム・前フレーム座標を補間
     └─ MainPixelShader（VelocityShader.usf）… Velocity = (CurPos - PrevPos) / 2 を書き出す

[3]  Nanite Velocity（Nanite メッシュ）
     └─ Nanite CS ベースのベロシティ生成（VisibilityBuffer から復元）
```

---

## 各ステップの詳細

### [1] Depth Only Pass

| 項目 | 内容 |
|-----|------|
| **概要** | Opaque は位置ストリームのみ。Masked はフルストリーム + Opacity テスト。PDO 対応 |
| **CPU 関数** | `FDeferredShadingSceneRenderer::RenderPrePassView()` |
| **シェーダー** | VS: `DepthOnlyVertexShader.usf#Main()`; PS: `DepthOnlyPixelShader.usf#Main()`（Masked のみ）|
| **出力** | `SceneDepthBuffer` |
| **CPU 詳細** | [[a_depth_prepass]] |
| **GPU シェーダー詳細** | [[detail_depth_only]] / [[ref_depth_only]] |

---

### [2〜3] Velocity Buffer

| 項目 | 内容 |
|-----|------|
| **概要** | スキン/静的/Nanite メッシュの現フレーム・前フレーム位置差分から Velocity を計算 |
| **CPU 関数** | `FDeferredShadingSceneRenderer::RenderVelocities()` |
| **シェーダー** | VS: `VelocityShader.usf#MainVertexShader()`; PS: `VelocityShader.usf#MainPixelShader()` |
| **出力** | `VelocityBuffer`（R16G16F）|
| **CPU 詳細** | [[b_velocity_buffer]] |
| **GPU シェーダー詳細** | [[detail_velocity]] / [[ref_velocity]] |

---

## シェーダー別 CPU 対応一覧

| シェーダーファイル | 主要エントリポイント | CPU 関数 | グループ |
|----------------|----------------|---------|---------|
| `DepthOnlyVertexShader.usf` | `Main()` | `RenderPrePassView()` | [[a_DepthOnly]] |
| `DepthOnlyPixelShader.usf` | `Main()` | `RenderPrePassView()`（Masked のみ）| [[a_DepthOnly]] |
| `VelocityShader.usf` | `MainVertexShader()` | `RenderVelocities()` | [[b_Velocity]] |
| `VelocityShader.usf` | `MainPixelShader()` | `RenderVelocities()` | [[b_Velocity]] |
