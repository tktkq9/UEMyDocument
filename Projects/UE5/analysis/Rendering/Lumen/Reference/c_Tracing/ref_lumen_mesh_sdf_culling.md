# リファレンス：LumenMeshSDFCulling.cpp

- グループ: c - Tracing
- 上位: [[c_lumen_tracing]]
- 関連: [[ref_lumen_tracing_utils]] | [[ref_lumen_heightfields]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenMeshSDFCulling.cpp`

---

## 概要

Lumen の Mesh SDF（Signed Distance Field）トレーシングに必要な**オブジェクトカリング**を実装するファイル。  
ビューのフラスタムグリッドに対して Mesh SDF オブジェクトと Heightfield を振り分け、  
シェーダー側でグリッドセルから近傍 SDF オブジェクトを O(1) で参照できるようにする。

---

## 主要関数

### SetupLumenMeshSDFTracingParameters

Mesh SDF トレーシングの共通パラメータを初期化する関数。  
DistanceField バッファとアトラスへの参照を `FLumenMeshSDFTracingParameters` にセットする。

```cpp
void SetupLumenMeshSDFTracingParameters(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    FLumenMeshSDFTracingParameters& OutParameters)
{
    // FDistanceFieldSceneData からオブジェクトバッファとアトラスパラメータを取得
    OutParameters.DistanceFieldObjectBuffers = DistanceField::SetupObjectBufferParameters(...);
    OutParameters.DistanceFieldAtlas         = DistanceField::SetupAtlasParameters(...);
    // CVar から各種スケール値を設定
    OutParameters.MeshSDFNotCoveredExpandSurfaceScale = GMeshSDFNotCoveredExpandSurfaceScale;
    OutParameters.MeshSDFNotCoveredMinStepScale       = GMeshSDFNotCoveredMinStepScale;
    OutParameters.MeshSDFDitheredTransparencyStepThreshold = GMeshSDFDitheredTransparencyStepThreshold;
}
```

### CullMeshObjectsToViewGrid

Mesh SDF オブジェクトを**フラスタムグリッドセルにカリング**する。  
Compute Shader で各グリッドセルに影響する SDF オブジェクトのインデックスリストを生成する。

```cpp
void CullMeshObjectsToViewGrid(
    const FViewInfo& View,
    const FScene* Scene,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    float MaxMeshSDFInfluenceRadius,    // SDF の影響最大半径
    float CardTraceEndDistanceFromCamera, // Surface Cache トレース最大距離
    int32 GridPixelsPerCellXY,          // グリッドセルのピクセルサイズ
    int32 GridSizeZ,                    // Z 方向のグリッド分割数
    FVector ZParams,                    // Z グリッドの深度パラメータ
    FRDGBuilder& GraphBuilder,
    FLumenMeshSDFGridParameters& OutGridParameters,
    ERDGPassFlags ComputePassFlags);
```

### CullHeightfieldObjectsForView

Heightfield（Landscape）オブジェクトをビューに対してカリングし、  
GPU バッファに有効な Heightfield インデックスリストを生成する。

```cpp
void CullHeightfieldObjectsForView(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    float MaxMeshSDFInfluenceRadius,
    float CardTraceEndDistanceFromCamera,
    FRDGBufferRef& NumCulledObjects,         // カリング後のオブジェクト数バッファ
    FRDGBufferRef& CulledObjectIndexBuffer); // カリング後のインデックスバッファ
```

### CullForCardTracing

Card（Surface Cache）向けのカリングをまとめて実行するラッパー関数。  
`CullMeshObjectsToViewGrid()` と `CullHeightfieldObjectsForView()` を内部で呼ぶ。

```cpp
void CullForCardTracing(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    const FLumenIndirectTracingParameters& IndirectTracingParameters,
    FLumenMeshSDFGridParameters& MeshSDFGridParameters,
    ERDGPassFlags ComputePassFlags);
```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.DiffuseIndirect.MeshSDF.AverageCulledCount` | 512 | 平均カリング後 Mesh SDF 数（バッファサイズのヒント）|
| `r.Lumen.DiffuseIndirect.MeshSDF.RadiusThreshold` | 30 | SDF を無視する最小オブジェクト半径（cm）|
| `r.Lumen.DiffuseIndirect.MeshSDF.NotCoveredExpandSurfaceScale` | 0.6 | 両面マテリアル SDF のサーフェス拡張スケール |
| `r.Lumen.DiffuseIndirect.MeshSDF.NotCoveredMinStepScale` | 32 | 両面マテリアル SDF の最小ステップスケール（高速化）|
| `r.Lumen.DiffuseIndirect.MeshSDF.DitheredTransparencyStepThreshold` | 0.1 | ディザ透明度の確率的スレッショルド |
| `r.LumenScene.Heightfield.CullForView` | 1 | Heightfield のビューカリング有効/無効 |
| `r.LumenScene.Heightfield.FroxelCulling` | 1 | Heightfield のフロクセルカリング有効/無効 |

---

## グリッドカリングの仕組み

```
CullMeshObjectsToViewGrid()
  │
  ├─ ビューフラスタムを GridPixelsPerCellXY × GridSizeZ のグリッドに分割
  │   例: 1920×1080 解像度 / 64px = 30×17 XY グリッド + 16 Z スライス
  │
  ├─ [Compute Pass] 各 Mesh SDF オブジェクトをグリッドセルにスキャッタ
  │   → オブジェクトの AABB がセルの錐台と交差するかチェック
  │   → 交差するセルの GridCulledMeshSDFObjectIndicesArray に追加
  │
  └─ 出力: FLumenMeshSDFGridParameters
        → シェーダー側: GridCulledMeshSDFObjectStartOffsetArray[CellIndex] で
          セルに属するオブジェクトの開始インデックスを取得
        → GridCulledMeshSDFObjectIndicesArray[StartIndex + i] でオブジェクトIDを取得
```

---

## NotCovered スケールの意味

```
Mesh SDF は単面ジオメトリを前提としているが、
草木など両面マテリアル（TwoSided）のメッシュでは
SDF の内側/外側の判定が曖昧になる。

NotCoveredExpandSurfaceScale = 0.6:
  → サーフェス付近で膨張判定を緩める（自己交差を防ぐ）

NotCoveredMinStepScale = 32:
  → 最小ステップを大きくして、薄い葉のジオメトリを高速スキップ
```
