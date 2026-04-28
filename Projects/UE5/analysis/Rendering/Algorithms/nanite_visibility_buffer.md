---
name: Nanite Visibility Buffer Rendering
description: Visibility Buffer (VisBuffer64) によるマテリアルシェーディング遅延と shading bin 振り分けの理論と UE 実装
type: project
---

# Nanite Visibility Buffer Rendering（マテリアル遅延シェーディング）

- 上位: [[_algorithm_index]]
- 関連: [[nanite_cluster_culling]] / [[nanite_sw_raster]] / [[nanite_lod]]
- 採用システム: Nanite の全マテリアル評価パス
- 出典 ID: **S20**（[[_source_index]]）— §6 Material Shading
- 影響元: Burns & Hunt 2013 "Visibility Buffer" HPG（原論文）

---

## 1. 何のためのアルゴリズムか

従来の forward / deferred ラスタは「三角形を描く時に **その三角形のマテリアル**を bind してフラグメントシェーダを走らせる」。しかし Nanite では:

- 1 ピクセル ≒ 1 三角形に近づく → ピクセル単位でマテリアルが切替
- マテリアル PSO 切替が膨大、PSO スイッチコストが支配的
- Pixel quad の `ddx/ddy` 微分も三角形またぎで破綻

### 素朴な手法の問題

- **マテリアルごとに drawcall**: PSO 切替コストで GPU が遊ぶ
- **G-Buffer に直接書込**: ピクセル単位でマテリアル分岐 → フラグメントシェーダのバラつき大
- **Forward Shading**: ライト数に比例して負荷増

### Nanite の貢献

- ラスタは **Visibility Buffer (VisBuffer64)** に「クラスタ ID + 三角形 ID」の 64-bit 値だけ書く
- **ShadingMask** で「このピクセルはどのマテリアルか」を示す
- 後段の **Shade Binning Pass** がピクセルをマテリアル単位にバケット振り分け
- **マテリアル単位の compute pass** でバケットを処理 → G-Buffer に出力
- → PSO 切替コスト最小化、マテリアル評価が完全並列

---

## 2. 理論

### 2.1 VisBuffer64 のレイアウト

64-bit / pixel:

| ビット | 内容 |
|-------|------|
| `[63:32]` (32-bit) | Depth（FP24 等で深度比較用） |
| `[31:7]` (25-bit) | VisibleClusterIndex（VisibleClustersSWHW へのインデックス） |
| `[6:0]` (7-bit) | TriangleIndex（クラスタ内の三角形 ID, max 128） |

```hlsl
// NaniteRasterizer.usf:652-653 抜粋（SW ラスタ）
uint PixelValue = (VisibleIndex + 1) << 7;
PixelValue |= TriIndex;
```

`VisibleIndex + 1` は 0 が「未書込」を意味するため。

### 2.2 SW / HW 共通の InterlockedMax

Visibility Buffer は **InterlockedMax(uint64)** で深度テストを兼ねる。SW ラスタも HW ラスタも同じ VisBuffer64 に書く（`NaniteWritePixel` 構造体）。深度を上位 32-bit に置くことで `InterlockedMax` 一発で「深度比較 + クラスタ/三角形 ID 書込」が完了。

### 2.3 マテリアルシェーディングパイプライン

```
[1] Cluster Culling → VisibleClustersSWHW
[2] Rasterize → VisBuffer64 + ShadingMask
[3] EmitMaterialDepth → DepthBuffer + ShadingMask
[4] ShadeBinning → MaterialBin（ピクセル → マテリアル ID）
[5] Per-Material CS → G-Buffer 出力
[6] Lighting / PostProcess（標準パス）
```

### 2.4 Shade Binning（マテリアル振り分け）

各ピクセルの `(VisibleClusterIndex, TriIndex)` から:
1. クラスタ ID → メッシュ → マテリアル ID を引く
2. ShadingBinData にマテリアル単位のピクセル座標を集計
3. 各マテリアル CS は「自分のバケットのピクセル座標リスト」だけ処理 → SIMD 効率最大

`ShadingBinData` 構造体は `FNaniteShadingUniformParameters::ShadingBinData`（`NaniteMaterials.cpp:54`）にバインド。

### 2.5 Quad-friendly ddx/ddy

VisibilityBuffer 上の隣接ピクセルが**同じクラスタ**であれば数学的に同じ三角形（高密度な Nanite では大半そう）。マテリアル CS は 2x2 quad で `ddx/ddy` を再計算可能。クラスタ境界でのみ微妙に degrade。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `NaniteMaterials.cpp` | Visibility Buffer 構築、Shade Binning、Material CS 起動 |
| `NaniteShading.cpp` | マテリアル単位 shading 実行 |
| `NaniteShared.cpp` | 共通ユニフォーム / 定数 |
| `NaniteShadeBinning.usf` | バケット集計 compute |
| `NaniteExportGBuffer.usf` | G-Buffer 出力 |
| `NaniteRasterizer.usf` | VisBuffer 書込本体（[[nanite_sw_raster]]） |
| `NaniteVertexFactory.h` | マテリアルラスタ用 VF |

### 3.2 Shading Uniform Parameters（`NaniteMaterials.cpp:45-66`）

```cpp
TRDGUniformBufferRef<FNaniteShadingUniformParameters> CreateDebugNaniteShadingUniformBuffer(FRDGBuilder& GraphBuilder)
{
    FNaniteShadingUniformParameters* UniformParameters = GraphBuilder.AllocParameters<FNaniteShadingUniformParameters>();

    UniformParameters->ClusterPageData      = Nanite::GStreamingManager.GetClusterPageDataSRV(GraphBuilder);
    UniformParameters->HierarchyBuffer      = Nanite::GStreamingManager.GetHierarchySRV(GraphBuilder);
    UniformParameters->VisibleClustersSWHW  = GraphBuilder.CreateSRV(...);
    UniformParameters->ShadingBinData       = GraphBuilder.CreateSRV(..., PF_R32_UINT);

    UniformParameters->VisBuffer64          = SystemTextures.Black;  // 実体は CullRaster pass が出力
    UniformParameters->DbgBuffer64          = SystemTextures.Black;
    UniformParameters->ShadingMask          = SystemTextures.Black;
    UniformParameters->MultiViewIndices     = ...;
    UniformParameters->MultiViewRectScaleOffsets = ...;
    UniformParameters->InViews              = ...;

    return GraphBuilder.CreateUniformBuffer(UniformParameters);
}
```

### 3.3 Raster Uniform Parameters（`NaniteMaterials.cpp:68-84`）

```cpp
TRDGUniformBufferRef<FNaniteRasterUniformParameters> CreateDebugNaniteRasterUniformBuffer(FRDGBuilder& GraphBuilder)
{
    FNaniteRasterUniformParameters* UniformParameters = GraphBuilder.AllocParameters<FNaniteRasterUniformParameters>();

    UniformParameters->PageConstants.X      = 0;
    UniformParameters->PageConstants.Y      = Nanite::GStreamingManager.GetMaxStreamingPages();
    UniformParameters->MaxNodes             = Nanite::FGlobalResources::GetMaxNodes();
    UniformParameters->MaxVisibleClusters   = Nanite::FGlobalResources::GetMaxVisibleClusters();
    UniformParameters->MaxCandidatePatches  = Nanite::FGlobalResources::GetMaxCandidatePatches();
    UniformParameters->MaxPatchesPerGroup   = 0u;
    UniformParameters->InvDiceRate          = 1.0f;
    UniformParameters->RenderFlags          = 0u;

    return GraphBuilder.CreateUniformBuffer(UniformParameters);
}
```

### 3.4 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.Nanite.MultipleSceneViewsInOnePass` | 1 | ISR 立体ビューを 1 パスで処理 |
| `r.Nanite.ShowStats` (`GNaniteShowStats`) | 0 | 統計表示 |

マテリアル評価本体は `NaniteShading.cpp` 側、バケット振り分けは `NaniteShadeBinning.usf` 側に CVar が分散。

### 3.5 マテリアル PSO 削減

従来 N マテリアル × M ライト = N×M PSO だったが、Nanite では:
- ラスタ PSO は **マテリアルラスタビン**単位（少数）
- シェーディング PSO は **マテリアル CS** 単位（マテリアル数だけ）
- ライティング/G-Buffer は標準 deferred と共有

---

## 4. 近似・省略の差分

| 項目 | 理想 | UE 実装 | 影響 |
|------|------|--------|------|
| MSAA | サンプル単位深度 | 1 サンプル/ピクセル（VisBuffer 単一値） | MSAA 非対応（TSR で代替） |
| Transparent | ソート + ブレンド | 不可（Visibility Buffer は不透明前提） | 半透明は別パス |
| Wireframe / 細線 | 三角形級 | 1 ピクセル未満は最近三角形に丸め | 細い線で aliasing |
| ddx/ddy | 三角形連続 | クラスタ境界で再計算 | 細部マテリアルパラメタの微分が不連続 |
| マテリアル切替 | drawcall | バケット集計 + CS | バケットサイズ偏りで GPU 効率劣化 |

---

## 5. パラメータと CVar

§3.4 にまとめ済み。`r.Nanite.ProgrammableRaster=0` でマテリアルラスタを無効化（Fixed-function path）。

---

## 6. 代替手法との比較

| 手法 | ID 表現 | マテリアル評価 | 採用 |
|------|--------|--------------|------|
| Forward Shading | フラグ単位 | drawcall 単位 PSO | UE Forward Renderer |
| Deferred Shading | G-Buffer | G-Buffer 上で uniform shading | UE Deferred 標準 |
| **Visibility Buffer (Burns/Hunt 2013)** | **PrimID** | **遅延 fetch + 遅延ラスタ** | **学術原典** |
| **Nanite Visibility Buffer** | **(ClusterID, TriID)** | **Shade Binning + Material CS** | **UE5 Nanite** |
| Tiled Forward+ | フラグ単位 | drawcall + tiled light cull | モバイル UE |

### Burns/Hunt VB vs Nanite VB

- 元論文: PrimID + バリセントリックを保存、後段で頂点フェッチ
- Nanite: ClusterID + TriID（クラスタストリーミングと統合）+ Shade Binning（マテリアル単位 CS）

---

## 7. 参考資料

- [S20] Karis / Stenson / Sherlock 2021 §6
- Burns / Hunt 2013 "The Visibility Buffer: A Cache-Friendly Approach to Deferred Shading"（HPG）
- 関連: [[nanite_cluster_culling]] / [[nanite_sw_raster]]

---

## 8. 相談用フック

- **理解度チェック**:
  - VisBuffer64 が 64-bit な理由 → 上位 32 bit を depth、下位 32 bit を ID にして InterlockedMax で 1 命令更新
  - Shade Binning の利点 → マテリアル単位 CS で完全並列、PSO 切替なし
  - 半透明非対応の理由 → Visibility Buffer は最前面 1 値しか保持できない
- **コード深掘り候補**:
  - `NaniteShadeBinning.usf` のバケット集計とディスパッチ引数生成
  - `NaniteExportGBuffer.usf` のマテリアル → G-Buffer 出力
  - `NaniteShading.cpp` のマテリアル毎 CS スケジューリング
- **未読箇所**:
  - S20 §6.3 ddx/ddy 復元の詳細式
  - Substrate との統合（`Substrate/Substrate.h`）
  - DBuffer デカール統合経路
- **次の派生**:
  - Substrate マテリアル → [[../Materials/...]]（未着手）
  - Nanite ライティング統合 → [[../Lighting/...]]（未着手）
  - Lumen Card Capture との統合 → [[lumen_surface_cache]]
