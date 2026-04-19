# GPU 処理：Modern パイプライン vs Legacy パイプライン

- 取得日: 2026-04-20
- 対象: `DeferredShadingRenderer.cpp` を中心としたフレーム実行順の差分
- 上位: [[01_gpu_overview]]
- 関連: [[01_nanite_gpu_overview]] / [[01_lumen_gpu_overview]] / [[01_vsm_gpu_overview]] / [[16_deferred_lighting_overview]]

---

## この記事の目的

`01_gpu_overview.md` は **Nanite + Lumen + VSM が全て有効な前提** で GPU 実行順を説明している。  
実際には各機能は CVar / ProjectSettings で独立に ON/OFF 可能で、**UE4 と同等の「古典 Deferred Shading」構成** でも UE5 は動作する。

本稿では:

- **Modern パイプライン**（Nanite + Lumen + VSM 全て ON）
- **Legacy パイプライン**（Nanite OFF + Lumen OFF + VSM OFF ≒ UE4 相当）

の 2 経路について **「どのステップが共通か」「どのステップがスキップ / 置換されるか」** を実行順に対比する。

> CVar を組み合わせて部分的に無効化する（例: Nanite だけ ON で Lumen OFF）現実的なケースは末尾の [組み合わせマトリクス](#組み合わせマトリクス) を参照。

---

## 2 つのパイプラインを分ける CVar

| 機能 | CVar | Modern | Legacy |
|------|------|--------|--------|
| Nanite メッシュ描画 | `r.Nanite=1` | ON | OFF |
| Nanite プロジェクト設定 | `ProjectSettings.Rendering.SupportNanite` | ON | OFF |
| 動的 GI | `r.DynamicGlobalIlluminationMethod` | `2`（Lumen） | `0`（None）or `1`（SSGI）|
| 反射 | `r.ReflectionMethod` | `2`（Lumen） | `0`（None）or `1`（SSR）|
| 仮想シャドウマップ | `r.Shadow.Virtual.Enable` | `1` | `0` |
| Shadow Map 方式 | `r.Shadow.Virtual.ForceOnLightTypes` | 既定（VSM 強制）| 既定（Cubemap / CSM） |

プロジェクト単位で切り替える場合は `DefaultEngine.ini` の `[SystemSettings]` に書くのが一般的。

---

## ステップ別 共通/差分 早見表

| # | ステップ | Modern | Legacy | 差分種別 |
|---|---------|--------|--------|---------|
| 1 | GPU Scene Update | ○ | ○ | 共通 |
| 2 | Shadow Depth | VSM Page 描画 + Nanite/非 Nanite | ShadowMap Atlas + Cubemap | **シェーダー置換** |
| 3 | Depth Pre-pass | 非 Nanite のみ | 全不透明 | **範囲拡大** |
| 4 | Nanite Rasterization | 実行 | **スキップ** | 完全スキップ |
| 5a | Lumen Card Capture | 実行 | **スキップ** | 完全スキップ |
| 5b | Lumen Surface Cache Lighting | 実行 | **スキップ** | 完全スキップ |
| 6 | Base Pass（GBuffer） | 非 Nanite のみ | 全不透明 | **範囲拡大** |
| 7a | Lumen Screen Probe Gather | AsyncCompute で実行 | **スキップ** | 完全スキップ |
| 7b | Lumen Reflections | AsyncCompute で実行 | **スキップ** | 完全スキップ |
| 8 | HZB 構築 | ○ | ○ | 共通 |
| 9 | Occlusion Queries | ○ | ○ | 共通 |
| 10 | Screen Space AO | Lumen Short Range AO | 古典 SSAO / GTAO | **シェーダー置換** |
| 11 | Direct Lighting | VSM Projection + Clustered/Standard | 2D ShadowMap Projection + Clustered/Standard | **シェーダー置換** |
| 12 | Indirect Composite | Lumen Final Composite（GI+反射） | **Reflection Environment + SSR + SkyLight** | **Pass 置換** |
| 13 | Sky / Atmosphere | ○ | ○ | 共通 |
| 14 | Fog（Height / Volumetric） | ○ | ○ | 共通 |
| 15 | Translucency | ○（TLV 入力が Lumen） | ○（TLV 入力が従来 IndirectLighting Cache） | **入力差のみ** |
| 16 | Post-processing | TSR 既定 | TAA 既定 | **設定差のみ** |

**要点:**

- **共通**: GPU Scene / HZB / Occlusion / Sky / Fog / Post-processing（骨格）
- **完全スキップ**: Nanite Raster / Lumen Card / Surface Cache Lighting / Screen Probe Gather / Lumen Reflections
- **置換**: Shadow Depth / SSAO / Direct Lighting の影サンプリング / Indirect Composite

---

## Modern パイプライン実行順

`01_gpu_overview.md` と同内容なので詳細はそちらを参照。骨格のみ再掲:

```
[1]  GPU Scene Update
[2]  Shadow Depth（VSM Page + Nanite/非 Nanite）
[3]  Depth Pre-pass（非 Nanite のみ）
[4]  Nanite Rasterization → VisBuffer → ShadeBinning → GBuffer Export
[5a] Lumen Card Capture
[5b] Lumen Surface Cache Lighting（Direct + Radiosity）
[6]  Base Pass（非 Nanite → GBuffer）
[7]  AsyncCompute: Lumen Screen Probe Gather + Lumen Reflections
[8]  HZB 構築
[9]  Occlusion Queries
[10] Lumen Short Range AO
[11] Direct Lighting（VSM Projection 経由で影サンプル）
[12] Lumen Final Composite（GI + 反射 + AO → SceneColor）
[13] Sky / Atmosphere
[14] Fog
[15] Translucency
[16] Post-processing（TSR → Bloom → Tonemap → ...）
```

---

## Legacy パイプライン実行順（UE4 相当）

```
[1]  GPU Scene Update                                  ← UE5 の機構だが Legacy でも実行
[2]  Shadow Depth Rendering
       ShadowDepth.usf:Main / MainPS
       → ShadowMap Atlas（方向光 CSM）+ CubeShadowMap（点光）
[3]  Depth Pre-pass
       DepthOnlyVertexShader.usf:Main
       → 全不透明の SceneDepthZ
[4]  ── Nanite 関連は全てスキップ ──
[5]  ── Lumen Surface Cache 関連は全てスキップ ──
[6]  Base Pass（全不透明 → GBuffer A/B/C/D + Velocity）
       BasePassVertexShader.usf / BasePassPixelShader.usf
[7]  ── Lumen AsyncCompute は全てスキップ ──
       ※ SSGI 有効時は [12] の直前で SSGI パスが走る
[8]  HZB 構築
[9]  Occlusion Queries
[10] Screen Space AO（古典 SSAO / GTAO）
       PostProcessAmbientOcclusion.usf:MainSetupPS / HorizonSearchCS / GTAOCombinedCS
[11] Direct Lighting（Deferred Lighting Pass）
       DeferredLightPixelShaders.usf:DeferredLightPixelShader
       + ShadowProjectionPixelShader.usf（2D ShadowMap サンプリング）
       + ShadowProjectionFromTranslucencyPixelShader.usf（半透明影）
[12] Indirect Composite（Lumen 置き換え群）
       [12-L1] SSGI（ScreenSpaceGI.usf）              ← r.DynamicGlobalIlluminationMethod=1 時
       [12-L2] Reflection Environment
                ReflectionEnvironmentPixelShader.usf
                → Reflection Capture Cubemap + SkyLight をサンプルして SceneColor に加算
       [12-L3] Screen Space Reflections（SSR）       ← r.ReflectionMethod=1 時
                ScreenSpaceReflections.usf
       [12-L4] Distance Field AO（任意）              ← r.AOGlobalDistanceField=1 時
                DistanceFieldScreenGridLighting.usf:AOLevelSetCS / ConeTraceCS
                → BentNormalAO を SceneColor に合成
[13] Sky / Atmosphere
       SkyAtmosphere.usf（Modern と同じ）
[14] Fog（Height + Volumetric Fog、Modern と同じ）
[15] Translucency
       BasePassPixelShaders.usf（Translucent permutation）
       TranslucentLightingShaders.usf（TLV は従来 IndirectLighting Cache から供給）
[16] Post-processing
       TAA（TemporalAA.usf）— 既定は TSR ではなく TAA
       + Bloom / Tonemap / DOF / MotionBlur（Modern と同じ）
```

---

## 主要な置換シェーダー対応表

| フェーズ | Modern シェーダー | Legacy シェーダー | CPU 関数（Modern / Legacy） |
|---------|-----------------|------------------|----------------------------|
| Shadow Depth | `NaniteShadow.usf:EmitShadowMapPS` + `ShadowDepth.usf:Main` | `ShadowDepth.usf:Main` のみ | `RenderVirtualShadowMaps()` / `RenderShadowDepthMaps()` |
| Opaque 描画 | `NaniteRasterizer.usf:HWRasterizeMS/PS` + `BasePassPixelShader.usf` | `BasePassPixelShader.usf`（全不透明） | `Nanite::RasterizePass()` + `RenderBasePass()` / `RenderBasePass()` のみ |
| 影サンプリング | `VirtualShadowMapProjection.usf:VirtualShadowMapProjection` | `ShadowProjectionPixelShader.usf:Main` | `RenderVirtualShadowMapProjection()` / `RenderShadowProjections()` |
| AO | `LumenScreenSpaceBentNormal.usf:ScreenSpaceShortRangeAOCS` | `PostProcessAmbientOcclusion.usf:GTAOCombinedCS` 等 | `RenderLumenShortRangeAO()` / `AddGTAOAmbientOcclusionPasses()` |
| 拡散 GI | `LumenScreenProbeTracing.usf` + `LumenScreenProbeGather.usf` | `ScreenSpaceGI.usf`（SSGI 時）/ IndirectLighting Cache | `RenderLumenFinalGather()` / `AddScreenSpaceDiffuseIndirectPass()` |
| 反射 | `LumenReflectionTracing.usf` + `LumenReflections.usf` | `ScreenSpaceReflections.usf` + `ReflectionEnvironmentPixelShader.usf` | `RenderLumenReflections()` / `RenderDeferredReflectionsAndSkyLighting()` |
| Indirect 合成 | `DiffuseIndirectComposite.usf:MainPS` | `ReflectionEnvironmentPixelShader.usf:ReflectionEnvironmentSkyLightingMain` | `RenderDiffuseIndirectAndAmbientOcclusion()` / `RenderDeferredReflectionsAndSkyLighting()` |
| SkyLight | Lumen が SkyLight 寄与を内包 | `SkyLightDiffuse.usf` + Reflection Environment に SkyLight Cubemap 加算 | — / `RenderDeferredReflectionsAndSkyLighting()` |

---

## VisBuffer と HZB の差分

Nanite の有無によって **GBuffer 書き込み方式** と **HZB の本数** が大きく変わる。

### Modern: VisBuffer 経由 + 2 種類の HZB

```
[4] Nanite Rasterization
    ├─ Cluster Culling Pass 1（前フレームの [8] HZB を利用）
    ├─ SW/HW Raster → VisBuffer64（中間バッファ）
    ├─ Nanite 内部 HZB 構築         ← Pass1 の VisBuffer から生成、Pass2 専用で使い捨て
    ├─ Cluster Culling Pass 2
    ├─ SW/HW Raster Pass 2 → VisBuffer64 に追記
    └─ GBuffer Export               ← VisBuffer を読んで GBuffer + ShadingMask に変換
[6] Base Pass（非 Nanite）          → 同じ GBuffer に追記
[8] パイプライン全体 HZB            ← 最終 SceneDepth から構築
                                      用途: Occlusion Query / SSR / Lumen Screen Probe / 次フレーム [4] Pass1
```

- VisBuffer64 は Nanite 専用の中間バッファ（64bit/pixel: Depth + ClusterID + TriID）。最終 GBuffer ではない。
- **GBuffer は Nanite GBuffer Export + 非 Nanite Base Pass の両方が書き込む**（同じ RenderTarget を共有）。
- **HZB は 2 種類**: Nanite 内部 HZB（使い捨て）とパイプライン全体 HZB（[8]）。後者は**次フレームの Nanite Pass1 に使われる**という循環構造。

### Legacy: VisBuffer なし + HZB は 1 種類

```
[3] Depth Pre-pass（全不透明）      → SceneDepthZ
[6] Base Pass（全不透明）            → GBuffer A/B/C/D + Velocity
[8] パイプライン全体 HZB            ← 最終 SceneDepth から構築
                                      用途: Occlusion Query / SSR（あれば）
```

- VisBuffer は存在しない。GBuffer は Base Pass が直接書き込む。
- HZB は [8] の 1 本のみ。Nanite がないので内部 HZB も循環も不要。
- Depth Pre-pass [3] が全不透明をカバーするため、Base Pass [6] は Depth Test のみで高速描画。

### 差分まとめ

| 項目 | Modern | Legacy |
|------|--------|--------|
| VisBuffer | あり（Nanite 専用中間バッファ） | なし |
| GBuffer 書き込み元 | Nanite GBuffer Export + 非 Nanite Base Pass | Base Pass のみ |
| HZB の本数 | 2 種類（内部 + 全体） | 1 種類（全体のみ） |
| HZB の循環 | あり（次フレーム [4] Pass1 に利用） | なし（用途は Occlusion Query / SSR） |

---

## 組み合わせマトリクス

現実のプロジェクトでは部分的に無効化することが多い。よくある組み合わせ:

| パターン | Nanite | Lumen GI | Lumen Refl | VSM | 典型用途 |
|---------|--------|---------|-----------|-----|---------|
| **Full Modern** | ON | ON | ON | ON | PC / 次世代コンソール向けの標準設定 |
| **Nanite only** | ON | OFF | OFF | ON | 静的ジオメトリ重視だが GI はライトマップ |
| **Lumen only** | OFF | ON | ON | OFF | 既存 UE4 資産 + 動的 GI だけ欲しい |
| **VSM only** | OFF | OFF | OFF | ON | VR / モバイル等で Nanite・Lumen が重い場合（稀）|
| **Full Legacy** | OFF | OFF | OFF | OFF | モバイル / VR / UE4 同等の描画コスト |

**組み合わせによる差分の読み方:**

- Nanite OFF にすると [4] が消え [3]/[6] が「全不透明」に拡大する。他は影響しない。
- Lumen GI OFF にすると [5a][5b][7a] がスキップされ、[10] が古典 SSAO、[12] が Reflection Environment + SSR に置換される。
- Lumen Refl だけ OFF（GI は ON）にすると [7b] がスキップされ、[12] 内で反射は SSR に置換、拡散 GI は Lumen のまま。
- VSM OFF にすると [2] が ShadowMap Atlas、[11] の影サンプリングが `ShadowProjectionPixelShader.usf` に置換される。Nanite は VSM なしでも動作する（Nanite Shadow Export の対象が 2D ShadowMap になる）。

---

## ue5-dive 起点

- 「Legacy の Reflection Environment 実装」 → `ReflectionEnvironmentPixelShader.usf:ReflectionEnvironmentSkyLightingMain` + `RenderDeferredReflectionsAndSkyLighting()`
- 「Legacy の SSR」 → `ScreenSpaceReflections.usf` + `ScreenSpaceReflections.cpp:RenderScreenSpaceReflections()`
- 「Legacy の 2D ShadowMap Projection」 → `ShadowProjectionPixelShader.usf:Main` + `ProjectedShadowInfo::RenderProjection()`
- 「Nanite が無効のときの分岐判定」 → `UseNanite()` / `r.Nanite` CVar / `Nanite::FGlobalResources`
- 「Lumen が無効のときの分岐判定」 → `ShouldRenderLumenDiffuseGI()` / `ShouldRenderLumenReflections()`
- 「Legacy Translucent Lighting Volume」 → `TranslucentLightingShaders.usf:FilterTranslucentVolumeCS` + 従来 IndirectLightingCache
