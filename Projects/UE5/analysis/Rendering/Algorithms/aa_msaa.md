---
name: MSAA (Forward+ / Mobile)
description: 標準ハードウェア Multi-Sample Anti-Aliasing。UE5 では Forward+ Renderer と Mobile 経路でのみ採用。Deferred 経路は GBuffer の互換性により非対応で、TAA/TSR が代替。
type: project
---

# MSAA（Forward+ / Mobile 専用）

- 出典 ID: **（標準）** ハードウェア機能（D3D/Vulkan/Metal の MultiSample 機構）
- UE 実装: `Engine/Source/Runtime/Renderer/Private/MobileShadingRenderer.cpp`, `ForwardShadingRenderer.cpp`, RHI 各実装
- ステータス: **完了 (2026-04-27)**
- 上位: [[_algorithm_index]] / [[../01_rendering_overview]]

---

## 1. 目的

ジオメトリエッジのジャギーをハードウェア機能で除去する。1 ピクセルあたり N サンプル位置で深度・ステンシル評価し、Pixel Shader は **カバレッジが変化したサブサンプル** でのみ実行（実質 1 回）→ Resolve で平均化。

UE 採用範囲:
- **Forward+ Renderer** （`r.ForwardShading=1`）
- **Mobile** （ES3.1 以上のタイル GPU）
- **VR**（Forward 経路で MSAA 推奨）

非採用:
- **Deferred Renderer**（GBuffer に複数サンプル分の Albedo/Normal を保持できないため）

---

## 2. 理論

### 2.1 サブサンプル分布

各ピクセルで N 個のサンプル位置（D3D 標準パターン）でカバレッジ評価。
- 2× MSAA: 2 サンプル
- 4× MSAA: 4 サンプル（ロータリー配置）
- 8× MSAA: 8 サンプル

Pixel Shader は **サンプルではなくフラグメント単位** で実行（Per-Sample Shading は別途指定）。

### 2.2 Resolve

最終的にサンプル平均で 1 ピクセル値を生成:

$$
C_\text{pixel} = \frac{1}{N}\sum_{i=1}^{N} C_i
$$

HDR の場合、線形空間で平均化しないと「白いエッジが暗くなる」アーティファクト。UE はトーンマップ前にカスタム Resolve（`MSAAHDRDecode`）で対処。

### 2.3 Forward+ との相性

Forward+ Tile Light Culling は Pixel Shader 単位なので、MSAA でカバレッジが変わってもライト計算は 1 回。コスト増は主に:
- Color/Depth ターゲットのメモリ帯域（× N）
- Resolve コスト
- Shader 実行は変わらない（フラグメント単位）

---

## 3. UE 実装

### 3.1 サンプル数取得

`GetDefaultMSAACount(FeatureLevel)`:
- Mobile: `r.MobileMSAA` (1/2/4/8、既定 4)
- Forward+: `r.MSAACount` (1/2/4/8、既定 4)

`MobileShadingRenderer.cpp:341`:
```cpp
NumMSAASamples = GetDefaultMSAACount(ERHIFeatureLevel::ES3_1);
```

### 3.2 RDG での扱い

Forward+ のレンダーターゲットは:
- `FRDGTextureMSAA` 構造体で MSAA + Resolved の 2 テクスチャを束ねて管理
- BasePass で MSAA テクスチャに書き込み
- ポストプロセス前に `ResolvePass` で 1× テクスチャに変換

### 3.3 Mobile 経路の特殊事情

タイル GPU（Adreno/Mali/Apple）では:
- **タイル内で MSAA 解決完了**（メモリ帯域節約）
- `GSupportsShaderFramebufferFetch` が true なら、Shader 内で Framebuffer Fetch して Resolve
- Metal で `MSAAHDRDecode` を併用（`PostProcessing.cpp:2563`）

### 3.4 Deferred で MSAA 非採用の理由

GBuffer は `RT0=BaseColor`, `RT1=Normal`, `RT2=PBR`, `RT3=...` の MRT。各サンプルで全部保持すると:
- メモリ消費 × N
- Pixel Shader（Lighting）でサンプル毎処理が必要
- Decal/Lighting/SSR/SSAO 等の後段処理がすべてサンプル対応必要

→ コストに見合わず、Deferred では TAA/TSR が代替。

---

## 4. 近似差分（理想 vs 実装）

| 項目 | 理想 SSAA | MSAA | 補足 |
|------|---------|------|------|
| AA 対象 | ジオメトリ + テクスチャ + シェーディング | ジオメトリ Edge のみ | Alpha Test/Alpha Coverage で部分対応 |
| Pixel Shader 実行 | サンプル毎（× N） | フラグメント毎（1 回） | 性能優位 |
| Subpixel Geometry | 厳密 | サンプル位置に依存 | 細い線で揺れ |
| 半透明 | 厳密 | A2C（Alpha to Coverage）で擬似対応 | ディザリング感 |
| Specular Aliasing | 解消 | **未対応**（マテリアル LOD/Roughness 補正で対処） | TAA は対応 |
| Temporal | なし | なし | TAA/TSR と排他 |
| HDR Resolve | 物理 | 線形空間 Resolve（UE 実装） | Default Resolve は誤差 |

---

## 5. 主要 CVar

| CVar | 既定 | 効果 |
|------|------|------|
| `r.MSAACount` | 4 | Forward+ MSAA サンプル数（1/2/4/8） |
| `r.MobileMSAA` | 4 | Mobile MSAA サンプル数 |
| `r.ForwardShading` | 0 | 1 で Forward+ Renderer 有効（MSAA 前提） |
| `r.AntiAliasingMethod` | 4 (TSR) | 0=None, 2=TAA, 3=MSAA, 4=TSR |
| `r.MSAA.CompositingSampleCount` | 4 | エディタ選択ガイド表示の MSAA |
| `vr.MobileMultiView` | 1 | VR で MSAA + Multi View |

`r.AntiAliasingMethod=3` は **Forward+ 経路でのみ有効**。Deferred で指定しても無視される。

---

## 6. 代替手法

| 手法 | 採用条件 | UE 実装 |
|------|---------|---------|
| **TAA (Karis)** | Deferred 標準 | [[aa_taa]] |
| **TSR** | UE 5.x 主流 | [[aa_tsr]] |
| **FXAA** | Mobile 軽量 | `r.AntiAliasingMethod=1`（旧） |
| **DLSS / FSR / XeSS** | プラグイン | サードパーティ |
| **SSAA**（手動） | Resolution Scale > 100% | `r.ScreenPercentage` |

---

## 7. 参考資料

- D3D 12 仕様書「Multisampling」
- **Akeley 1993** "Reality Engine Graphics" SIGGRAPH 1993 → MSAA 元アイデア
- **Persson 2008** "ATI Radeon HD 2000 programming guide" → A2C 詳細
- 出典 ID 標準（[[_source_index]] では論文記載なし）

---

## 8. 相談用フック（不確かなポイント）

- **Specular Aliasing 非対応**: MSAA は形状エッジのみ。マテリアル内の輝点・ハイライトのエイリアシングは別途 Roughness Filtering（[Toksvig 2005] / Specular AA）で対処する必要があり、UE では `r.NormalCurvatureToRoughness` 等が関連 CVar。
- **Mobile MSAA の実コスト**: タイル GPU では帯域コストはほぼゼロ（オンチップ Resolve）。一方 Desktop Forward+ では帯域 × N でコストが顕著。
- **エディタの MSAA**: ビューポート編集中の Editor Primitive レイヤーは `r.MSAA.CompositingSampleCount` で別管理。ゲームプレイビューと異なる場合あり。
- **VR + MSAA**: VR では SSAA（Resolution Scale）よりも MSAA + Multi View が一般的。Stereo Rendering で MSAA は概ね必須。
- **Alpha to Coverage**: 葉や髪の毛等の Alpha Test マテリアルは A2C（`MaterialSettings.bUseAlphaToCoverage`）で MSAA サブサンプル単位の擬似透明化。MSAA 非対応経路ではディザリング劣化。

---

## 関連ドキュメント

- [[aa_taa]] / [[aa_tsr]] — Deferred 経路の代替 AA
- [[../01_rendering_overview]]
