# Rendering Algorithms ソース索引

外部資料（論文・SIGGRAPH course notes・公式記事）のメタデータと UE 実装インデックス。

- 上位: [[_algorithm_index]]
- 更新日: 2026-04-29（21 PDF キャッシュ完了 / 残 S21 は公式論文不在）
- ダウンロード補助: [[_papers/_download.sh]]

---

## 1. 取得済み外部資料

`URL` は取得元、`ローカル` は `_papers/` 配下のキャッシュ（`.gitignore` で除外）。  
ローカル列が空の項目は未取得。Step 3（資料収集）で順次埋める。

### BRDF / シェーディング

| ID | タイトル | 著者 | 年 | 種別 | URL | ローカル | 重要箇所 |
|----|--------|-----|----|----|-----|--------|--------|
| S01 | Real Shading in Unreal Engine 4 | Karis (Epic) | 2013 | SIGGRAPH course | https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf | `_papers/S01_Karis_RealShadingUE4_2013.pdf` | slide 11 NDF, slide 12 G, slide 14 IBL Split-Sum |
| S02 | Microfacet Models for Refraction through Rough Surfaces | Walter et al. | 2007 | EGSR | https://www.graphics.cornell.edu/~bjw/microfacetbsdf.pdf | `_papers/S02_Walter_Microfacet_2007.pdf` | Eq.33 GGX NDF, Eq.34 Smith G |
| S03 | Physically-Based Shading at Disney | Burley | 2012 | SIGGRAPH course | https://media.disneyanimation.com/uploads/production/publication_asset/48/asset/s2012_pbs_disney_brdf_notes_v3.pdf | `_papers/S03_Burley_PBS_Disney_2012.pdf` | §5.3 Disney Diffuse, §5.4 GGX 採用根拠 |
| S03b | Approximate Reflectance Profiles for Efficient SSS | Burley, Christensen | 2015 | Disney SIGGRAPH | https://web.archive.org/web/2023/https://graphics.pixar.com/library/ApproxBSSRDF/paper.pdf | `_papers/S03b_Christensen_Burley_NormalizedSSS_2015.pdf` | Burley Normalized SSS（[[sss_burley]] 参照） |
| S03c | Physically Based Hair Shading in Unreal | Karis (Epic) | 2016 | SIGGRAPH course | https://blog.selfshadow.com/publications/s2016-shading-course/karis/s2016_pbs_epic_hair.pdf | `_papers/S03_Karis_HairShading_2016.pdf` | 異方性 GGX 拡張（[[brdf_ggx_aniso]] 参照） |
| S04 | Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs | Heitz | 2014 | JCGT | https://jcgt.org/published/0003/02/03/paper.pdf | `_papers/S04_Heitz_MaskingShadowing_2014.pdf` | §5 Smith G2, Joint vs Separable |
| S05 | An Inexpensive BRDF Model for Physically-Based Rendering | Schlick | 1994 | Eurographics | https://onlinelibrary.wiley.com/doi/10.1111/1467-8659.1330233 | `_papers/S05_Schlick_1994.pdf`（PDF不可・要旨のみ） | Fresnel 5乗近似 |
| S06 | Getting More Physical in Call of Duty: Black Ops II | Lazarov | 2013 | SIGGRAPH course | https://blog.selfshadow.com/publications/s2013-shading-course/lazarov/s2013_pbs_black_ops_2_notes.pdf | `_papers/S06_Lazarov_BlackOps2_2013.pdf` | EnvBRDF 多項式近似 |
| S07 | A Microfacet Based Coupled Specular-Matte BRDF Model with Importance Sampling | Kelemen | 2001 | Eurographics short | https://diglib.eg.org/bitstreams/17596c08-140e-46cf-bf8d-57113f201d38/download | `_papers/S07_Kelemen_SpecularMatte_2001.pdf` | Vis_Kelemen 簡略化 G |
| S08 | Real-Time Polygonal-Light Shading with Linearly Transformed Cosines | Heitz et al. | 2016 | SIGGRAPH | https://eheitzresearch.wordpress.com/415-2/ | （Google Drive 配信のため curl 不適 / 手動取得要） | LTC 行列の構築方法 |

### グローバルイルミネーション

| ID | タイトル | 著者 | 年 | 種別 | URL | ローカル | 重要箇所 |
|----|--------|-----|----|----|-----|--------|--------|
| S10 | Lumen: Real-time Global Illumination in Unreal Engine 5 | Wright, Heitz, Hillaire (Epic) | 2022 | SIGGRAPH course | https://advances.realtimerendering.com/s2022/SIGGRAPH2022-Advances-Lumen-Wright%20et%20al.pdf | `_papers/S10_Wright_Lumen_2022.pdf` | Surface Cache 章, Final Gather 章 |
| S11 | ReSTIR GI: Path Resampling for Real-Time Path Tracing | Ouyang et al. | 2021 | EGSR | https://d1qx31qr3h6wln.cloudfront.net/publications/ReSTIR%20GI.pdf | `_papers/S11_Ouyang_ReSTIR_GI_2021.pdf`（158MB） | Spatiotemporal reservoir resampling |

### ジオメトリ

| ID | タイトル | 著者 | 年 | 種別 | URL | ローカル | 重要箇所 |
|----|--------|-----|----|----|-----|--------|--------|
| S20 | Nanite: A Deep Dive | Karis, Stubbe, Wihlidal (Epic) | 2021 | SIGGRAPH course | https://advances.realtimerendering.com/s2021/Karis_Nanite_SIGGRAPH_Advances_2021_final.pdf | `_papers/S20_Karis_Nanite_2021.pdf` | DAG 構築, Visibility Buffer, SW Rasterizer |

### シャドウ

| ID | タイトル | 著者 | 年 | 種別 | URL | ローカル | 重要箇所 |
|----|--------|-----|----|----|-----|--------|--------|
| S21 | Virtual Shadow Maps in Fortnite Battle Royale Chapter 4 | Karis (Epic) | 2023 | Unreal Fest / Epic 公式 | https://dev.epicgames.com/documentation/en-us/unreal-engine/virtual-shadow-maps-in-unreal-engine | （公式論文/スライド PDF 不在 / UE5 ドキュメントが一次情報源） | Sparse Page allocation, Per-page caching |
| S22 | Percentage-Closer Soft Shadows | Fernando | 2005 | SIGGRAPH sketch | http://developer.download.nvidia.com/shaderlibrary/docs/shadow_PCSS.pdf | `_papers/S22_Fernando_PCSS_2005.pdf` | Blocker search → Penumbra 推定 |

### アンチエイリアス / 解像度

| ID | タイトル | 著者 | 年 | 種別 | URL | ローカル | 重要箇所 |
|----|--------|-----|----|----|-----|--------|--------|
| S30 | High Quality Temporal Supersampling | Karis (Epic) | 2014 | SIGGRAPH course | https://advances.realtimerendering.com/s2014/epic/TemporalAA.pptx | （PPTX のみ・27MB / 旧 PDF CDN は 403） | Variance Clipping, YCoCg |
| S31 | Temporal Super Resolution in Unreal Engine 5 | Epic | 2022 | Unreal Fest | https://assets-unreal2-epic-prod-us2.s3.dualstack.us-east-1.amazonaws.com/original/4X/6/6/d/66dab26619dbce6c73ad3cf02722e1b1787f0cc9.pdf | `_papers/S31_Epic_TSR_2022.pdf` | History rectification, dilation |

### スクリーンスペース

| ID | タイトル | 著者 | 年 | 種別 | URL | ローカル | 重要箇所 |
|----|--------|-----|----|----|-----|--------|--------|
| S40 | Scalable Ambient Obscurance | McGuire et al. | 2012 | HPG | https://research.nvidia.com/sites/default/files/pubs/2012-06_Scalable-Ambient-Obscurance/McGuire12SAO.pdf | `_papers/S40_McGuire_SAO_2012.pdf` | HBAO の派生 |
| S41 | Stochastic Screen-Space Reflections | Stachowiak (DICE) | 2015 | SIGGRAPH course | https://advances.realtimerendering.com/s2015/ | （advances 2015 は PPT 58MB のみ / PDF 配信なし） | レイマーチ + リサンプル |
| S42 | Deep G-Buffers for Stable Global Illumination Approximation | Mara et al. | 2016 | I3D | https://casual-effects.com/research/Mara2016DeepGBuffer/Mara2016DeepGBuffer.pdf | `_papers/S42_Mara_DeepGBuffer_2016.pdf` | 2 層 G-buffer GI |

### ポストプロセス

| ID | タイトル | 著者 | 年 | 種別 | URL | ローカル | 重要箇所 |
|----|--------|-----|----|----|-----|--------|--------|
| S50 | Academy Color Encoding System | AMPAS | 2014- | 規格 | https://acescentral.com/ | （未取得） | RRT + ODT, 公式 CTL |
| S51 | Photographic Tone Reproduction for Digital Images | Reinhard et al. | 2002 | SIGGRAPH | https://www-old.cs.utah.edu/docs/techreports/2002/pdf/UUCS-02-001.pdf | `_papers/S51_Reinhard_PhotographicTone_2002.pdf` | Reinhard tone curve 原典 |
| S51b | Filmic Tonemapping with Piecewise Power Curves | Hable | 2017 | ブログ | http://filmicworlds.com/blog/filmic-tonemapping-with-piecewise-power-curves/ | （ブログ記事 / PDF 配信なし） | Hable Filmic 形式 |
| S52 | Next Generation Post Processing in Call of Duty: Advanced Warfare | Jimenez | 2014 | SIGGRAPH course | https://www.iryoku.com/next-generation-post-processing-in-call-of-duty-advanced-warfare/ | （PPTX のみ・407MB / PDF 配信なし） | Mip-pyramid Bloom 詳細 |
| S53 | Practical Real-Time Strategies for Accurate Indirect Occlusion | Jimenez | 2016 | SIGGRAPH course | https://blog.selfshadow.com/publications/s2016-shading-course/activision/s2016_pbs_activision_occlusion.pdf | `_papers/S53_Jimenez_GTAO_2016.pdf` | GTAO（FFT Bloom 別資料） |
| S54 | A Life of a Bokeh | Abadie (DICE) | 2018 | SIGGRAPH course | https://advances.realtimerendering.com/s2018/2018-08-13-SIGGRAPH-A-Life-of-a-Bokeh.pptx | （PPTX のみ・150MB / PDF 配信なし） | DOF カーネル |
| S55 | A Reconstruction Filter for Plausible Motion Blur | McGuire et al. | 2012 | I3D | https://casual-effects.com/research/McGuire2012Blur/McGuire12Blur.pdf | `_papers/S55_McGuire_MotionBlur_2012.pdf` | Tile-based scatter-as-gather |

### 大気・体積

| ID | タイトル | 著者 | 年 | 種別 | URL | ローカル | 重要箇所 |
|----|--------|-----|----|----|-----|--------|--------|
| S60 | A Scalable and Production Ready Sky and Atmosphere Rendering Technique | Hillaire (Epic) | 2020 | EGSR | https://sebh.github.io/publications/egsr2020.pdf | `_papers/S60_Hillaire_SkyAtmosphere_2020.pdf` | LUT ベース散乱 |
| S61 | The Real-Time Volumetric Cloudscapes of Horizon: Zero Dawn | Schneider | 2015 | SIGGRAPH course | https://d3d3g8mu99pzk9.cloudfront.net/AndrewSchneider/The-Real-time-Volumetric-Cloudscapes-of-Horizon-Zero-Dawn.pdf | `_papers/S61_Schneider_Clouds_2015.pdf` | Worley Noise 雲, レイマーチ |
| S62 | Physically Based and Unified Volumetric Rendering in Frostbite | Hillaire (DICE) | 2015 | SIGGRAPH course | https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite | （RAR アーカイブ URL 失効 / 要再調査） | Volumetric Fog の体積積分 |

### SSS

| ID | タイトル | 著者 | 年 | 種別 | URL | ローカル | 重要箇所 |
|----|--------|-----|----|----|-----|--------|--------|
| S70 | Separable Subsurface Scattering | Jimenez et al. | 2015 | EGSR / CGF | https://users.cg.tuwien.ac.at/zsolnai/wp/wp-content/uploads/2014/12/ssss.pdf | `_papers/S70_Jimenez_SeparableSSS_2015.pdf` | カーネル分離 |

---

## 2. UE 実装インデックス（数式関連ファイル）

各システム内で「アルゴリズム的な核」となるシェーダ/CPP ファイルとその主要関数。

### BRDF.ush（`Engine/Shaders/Private/BRDF.ush`）

| 行 | 関数 | アルゴリズム ID |
|----|------|---------------|
| 161 | `Diffuse_Lambert` | （古典） |
| 183 | `Diffuse_Burley` | S03 |
| 331 | `D_GGX` | S02 |
| 339 | `D_GGXaniso` | S03 |
| 366 | `Vis_Kelemen` | S07 |
| 374 | `Vis_Schlick` | S05 |
| 392 | `Vis_SmithJointApprox` | S04 |
| 401 | `Vis_SmithJoint` (full) | S04 |
| 423 | `F_Schlick` | S05 |
| 611 | `EnvBRDFApprox` (Karis Split-Sum) | S01 |
| 630 | `EnvBRDFApproxLazarov` | S06 |
| 705 | `D_InvGGX` (Cloth) | （Epic 独自） |
| 792 | `ConvertRoughnessGGXToBeckmann` | （変換式） |

### ShadingModels.ush

各 ShadingModel（DefaultLit / Subsurface / Cloth / Hair / Eye / Cloth / ClearCoat 等）で `F_Schlick` `Diffuse_Burley` `Vis_SmithJointApprox` を組み合わせる。詳細は [[brdf_disney_diffuse]] 等で。

### Lumen（`Engine/Source/Runtime/Renderer/Private/Lumen/`）

| ファイル | 担当 | アルゴリズム ID |
|---------|------|---------------|
| `LumenSceneCardCapture.cpp` | カード生成・キャプチャ | S10 |
| `LumenSurfaceCache.cpp` (※実体は分散) | Surface Cache 管理 | S10 |
| `LumenRadianceCache.cpp` | Radiance Cache | S10 |
| `LumenMeshSDFCulling.cpp` | SW RT カリング | S10 |
| `LumenHardwareRayTracingCommon.cpp` | HW RT 共通 | S10 |
| `LumenScreenProbeGather.cpp` | Final Gather | S10 |
| `LumenReSTIRGather.cpp` | ReSTIR Gather | S10, S11 |

### Nanite（`Engine/Source/Runtime/Renderer/Private/Nanite/`）

| ファイル | 担当 | アルゴリズム ID |
|---------|------|---------------|
| `NaniteCullRaster.cpp` | DAG Cluster Culling | S20 |
| `NaniteMaterials.cpp` | Visibility Buffer Material | S20 |
| シェーダ `NaniteRasterizer.usf` | SW Rasterizer | S20 |

### VSM（`Engine/Source/Runtime/Renderer/Private/VirtualShadowMaps/`）

| ファイル | 担当 | アルゴリズム ID |
|---------|------|---------------|
| `VirtualShadowMapArray.cpp` | Sparse Page アロケーション | S21 |
| `VirtualShadowMapCacheManager.cpp` | Per-page caching | S21 |
| `VirtualShadowMapClipmap.cpp` | Directional Light Clipmap | S21 |
| `VirtualShadowMapProjection.cpp` | サンプリング | S21 |

---

## 3. 未解決事項（相談トリガー）

セッション中に発生した質問を蓄積する。解決後はステータス更新（削除しない）。

| 項目 | 個別ドキュメント | 質問内容 | ステータス |
|------|---------------|--------|--------|
| `a2 = a*a` の意味 | [[brdf_ggx]] | UE は `Roughness*Roughness` を α として使うが、Disney（S03）が推奨した「視覚的線形性」の根拠の数式詳細 | 未解決 |
| Smith-Joint Approx の誤差 | [[brdf_smith]] | Joint の正確な定義、フル形式との誤差プロット | 未解決 |
| Karis Split-Sum 第二項 | [[brdf_split_sum]] | 環境マップを Roughness で前畳み込みする式の導出 | 未解決 |
| GGX α と Roughness の関係 | [[brdf_ggx]] | 「α = Roughness²」採用の歴史的経緯（UE3 → UE4 移行時） | 未解決 |
| Lazarov 多項式の係数 | [[brdf_lazarov_env]] | 4 次多項式フィッティングの根拠と元データ | 未解決 |
| Lumen Surface Cache の解像度 | [[lumen_surface_cache]] | カード解像度の選択基準（メッシュ面積依存？） | 未解決 |
| Nanite クラスタサイズ 128 | [[nanite_cluster_culling]] | なぜ 1 クラスタ 128 三角形なのか、ハードウェア依存性 | 未解決 |

---

## 4. メンテナンス指針

- 新しい論文を取得したら `_papers/` に保存し本表「ローカル」列に追記
- 「相談用フック」で挙がった疑問は本表「未解決事項」に転記
- 解決した項目はステータスを更新（削除しない、後で振り返れるように）
- 個別ドキュメントを作成したら `_algorithm_index.md` のステータスも更新
