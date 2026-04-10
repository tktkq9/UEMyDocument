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

## SetupLumenMeshSDFTracingParameters

Mesh SDF トレーシングの共通パラメータを初期化する関数。  
DistanceField バッファとアトラスへの参照を `FLumenMeshSDFTracingParameters` にセットする。

```cpp
void SetupLumenMeshSDFTracingParameters(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene,
    const FViewInfo& View,
    FLumenMeshSDFTracingParameters& OutParameters);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン（FDistanceFieldSceneData を取得するため）|
| `View` | `const FViewInfo&` | 現在のビュー |
| `OutParameters` | `FLumenMeshSDFTracingParameters&` | 出力パラメータ構造体 |

### 内部処理フロー

1. **DistanceField バッファの取得**
   ```cpp
   // FDistanceFieldSceneData からオブジェクトバッファとアトラスパラメータを取得
   OutParameters.DistanceFieldObjectBuffers =
       DistanceField::SetupObjectBufferParameters(GraphBuilder, Scene->DistanceFieldSceneData);
   OutParameters.DistanceFieldAtlas =
       DistanceField::SetupAtlasParameters(GraphBuilder, Scene->DistanceFieldSceneData);
   ```

2. **CVar からスケール値をセット**
   ```cpp
   OutParameters.MeshSDFNotCoveredExpandSurfaceScale =
       GMeshSDFNotCoveredExpandSurfaceScale;   // デフォルト 0.6
   OutParameters.MeshSDFNotCoveredMinStepScale =
       GMeshSDFNotCoveredMinStepScale;         // デフォルト 32
   OutParameters.MeshSDFDitheredTransparencyStepThreshold =
       GMeshSDFDitheredTransparencyStepThreshold; // デフォルト 0.1
   ```

### 使用箇所

- [[ref_lumen_mesh_sdf_culling]] — `CullMeshObjectsToViewGrid()` の直前に呼ばれ、基本パラメータを確保する
- [[ref_lumen_screen_probe_tracing]] — Screen Probe SDF トレース前のパラメータ初期化

---

## CullMeshObjectsToViewGrid

Mesh SDF オブジェクトを**フラスタムグリッドセルにカリング**する。  
Compute Shader で各グリッドセルに影響する SDF オブジェクトのインデックスリストを生成する。

```cpp
void CullMeshObjectsToViewGrid(
    const FViewInfo& View,
    const FScene* Scene,
    const FLumenSceneFrameTemporaries& FrameTemporaries,
    float MaxMeshSDFInfluenceRadius,
    float CardTraceEndDistanceFromCamera,
    int32 GridPixelsPerCellXY,
    int32 GridSizeZ,
    FVector ZParams,
    FRDGBuilder& GraphBuilder,
    FLumenMeshSDFGridParameters& OutGridParameters,
    ERDGPassFlags ComputePassFlags);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `View` | `const FViewInfo&` | 現在のビュー（フラスタム・解像度の取得用）|
| `Scene` | `const FScene*` | SDF オブジェクトリストの取得源 |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | フレームバッファ参照 |
| `MaxMeshSDFInfluenceRadius` | `float` | SDF オブジェクトの最大影響半径（cm）|
| `CardTraceEndDistanceFromCamera` | `float` | Surface Cache トレースの最大カメラ距離（cm）|
| `GridPixelsPerCellXY` | `int32` | XY グリッドセルのピクセルサイズ（例: 64）|
| `GridSizeZ` | `int32` | Z 方向のグリッド分割数（例: 16）|
| `ZParams` | `FVector` | Z スライスの深度マッピングパラメータ |
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `OutGridParameters` | `FLumenMeshSDFGridParameters&` | 出力カリング結果（シェーダーパラメータ）|
| `ComputePassFlags` | `ERDGPassFlags` | Compute パスフラグ（Async Compute 対応用）|

### 内部処理フロー

1. **グリッドサイズの計算**
   ```cpp
   // 例: 1920×1080 / 64px = 30×17 セル（XY）× 16 スライス（Z）
   FIntVector GridSize;
   GridSize.X = FMath::DivideAndRoundUp(View.ViewRect.Width(),  GridPixelsPerCellXY);
   GridSize.Y = FMath::DivideAndRoundUp(View.ViewRect.Height(), GridPixelsPerCellXY);
   GridSize.Z = GridSizeZ;
   ```

2. **バッファのアロケート**
   ```cpp
   // カウンタ・オフセット・インデックスバッファを RDG で確保
   FRDGBufferRef NumGridCulledMeshSDFObjects =
       GraphBuilder.CreateBuffer(..., GridSize.X * GridSize.Y * GridSize.Z);
   FRDGBufferRef GridCulledMeshSDFObjectIndicesArray = GraphBuilder.CreateBuffer(...);
   ```

3. **カリング Compute Pass（スキャッタ方式）**
   ```cpp
   // FLumenMeshSDFCullObjectsToGridCS: 各 SDF オブジェクトの AABB を
   // グリッドセルの錐台（Froxel）と交差判定してインデックスをスキャッタ
   TShaderRef<FLumenMeshSDFCullObjectsToGridCS> ComputeShader = ...;
   FComputeShaderUtils::AddPass(GraphBuilder, ..., ComputeShader, PassParameters,
       FIntVector(NumMeshSDFObjects, 1, 1));
   ```

4. **Prefix Sum（オフセット配列の構築）**
   ```cpp
   // 各セルのオブジェクト数から開始オフセット配列を生成
   // NumGridCulledMeshSDFObjects → GridCulledMeshSDFObjectStartOffsetArray
   AddPrefixSumPass(GraphBuilder, NumGridCulledMeshSDFObjects, GridCulledMeshSDFObjectStartOffsetArray);
   ```

5. **アウトパラメータへのバインド**
   ```cpp
   OutGridParameters.NumGridCulledMeshSDFObjects = GraphBuilder.CreateSRV(NumGridCulledMeshSDFObjects);
   OutGridParameters.GridCulledMeshSDFObjectStartOffsetArray = ...;
   OutGridParameters.GridCulledMeshSDFObjectIndicesArray = ...;
   OutGridParameters.CardGridPixelSizeShift = FMath::FloorLog2(GridPixelsPerCellXY);
   OutGridParameters.CardGridZParams        = ZParams;
   OutGridParameters.CullGridSize           = GridSize;
   ```

### 使用箇所

- [[ref_lumen_mesh_sdf_culling]] — `CullForCardTracing()` の内部で呼ばれる
- [[ref_lumen_screen_probe_tracing]] — Screen Probe トレースパスの準備として直接呼ばれる場合もある

---

## CullHeightfieldObjectsForView

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
    FRDGBufferRef& NumCulledObjects,
    FRDGBufferRef& CulledObjectIndexBuffer);
```

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | Heightfield リストの取得源 |
| `View` | `const FViewInfo&` | 現在のビュー |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | フレームバッファ参照 |
| `MaxMeshSDFInfluenceRadius` | `float` | 影響最大半径 |
| `CardTraceEndDistanceFromCamera` | `float` | Surface Cache の最大距離 |
| `NumCulledObjects` | `FRDGBufferRef&` | 出力：カリング後の Heightfield 数バッファ |
| `CulledObjectIndexBuffer` | `FRDGBufferRef&` | 出力：カリング後のインデックスバッファ |

### 内部処理フロー

1. **フラスタムとの交差判定**
   ```cpp
   // CPU 側でビューフラスタムと Heightfield の AABB を交差チェック
   // r.LumenScene.Heightfield.CullForView = 1 の場合のみ実行
   for (const FLumenHeightfield& Heightfield : LumenSceneData.Heightfields) {
       if (IsVisible(Heightfield, View)) {
           CulledIndices.Add(Heightfield.GPUIndex);
       }
   }
   ```

2. **GPU バッファへアップロード**
   ```cpp
   NumCulledObjects         = CreateAndUploadBuffer(GraphBuilder, CulledCount);
   CulledObjectIndexBuffer  = CreateAndUploadBuffer(GraphBuilder, CulledIndices);
   ```

### 使用箇所

- [[ref_lumen_mesh_sdf_culling]] — `CullForCardTracing()` から内部呼び出し。結果は `FLumenMeshSDFGridParameters` に格納される

---

## CullForCardTracing

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

### パラメータ

| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー |
| `Scene` | `const FScene*` | シーン参照 |
| `View` | `const FViewInfo&` | 現在のビュー |
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | フレームバッファ参照 |
| `IndirectTracingParameters` | `const FLumenIndirectTracingParameters&` | トレース距離パラメータ（カリング範囲の決定に使用）|
| `MeshSDFGridParameters` | `FLumenMeshSDFGridParameters&` | 出力：SDF + Heightfield カリング結果 |
| `ComputePassFlags` | `ERDGPassFlags` | Compute パスフラグ |

### 内部処理フロー

```cpp
// 1. 基本 SDF パラメータのセットアップ
SetupLumenMeshSDFTracingParameters(GraphBuilder, Scene, View,
    MeshSDFGridParameters.TracingParameters);

// 2. Heightfield をビューカリング
FRDGBufferRef NumCulledHeightfields, CulledHeightfieldIndexBuffer;
CullHeightfieldObjectsForView(GraphBuilder, Scene, View, FrameTemporaries,
    MaxInfluenceRadius, IndirectTracingParameters.CardTraceEndDistanceFromCamera,
    NumCulledHeightfields, CulledHeightfieldIndexBuffer);

// 3. Mesh SDF をグリッドカリング（Heightfield も同グリッドへ）
CullMeshObjectsToViewGrid(View, Scene, FrameTemporaries,
    MaxInfluenceRadius, IndirectTracingParameters.CardTraceEndDistanceFromCamera,
    GridPixelsPerCellXY, GridSizeZ, ZParams,
    GraphBuilder, MeshSDFGridParameters, ComputePassFlags);

// 4. Heightfield グリッドカリング結果を MeshSDFGridParameters に格納
MeshSDFGridParameters.NumCulledHeightfieldObjects = ...;
MeshSDFGridParameters.CulledHeightfieldObjectIndexBuffer = ...;
```

### 使用箇所

- [[ref_lumen_scene_lighting]] — `RenderLumenSceneLighting()` の Direct Lighting パスで Surface Cache トレース用にカリング
- [[ref_lumen_screen_probe_tracing]] — Screen Probe SDF トレースパスの直前で呼ばれる
- [[ref_lumen_radiosity]] — Radiosity プローブトレースの前にカリング

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.Lumen.DiffuseIndirect.MeshSDF.AverageCulledCount` | 512 | 平均カリング後 Mesh SDF 数（バッファサイズのヒント）|
| `r.Lumen.DiffuseIndirect.MeshSDF.RadiusThreshold` | 30 | SDF を無視する最小オブジェクト半径（cm）|
| `r.Lumen.DiffuseIndirect.MeshSDF.NotCoveredExpandSurfaceScale` | 0.6 | 両面マテリアル SDF のサーフェス拡張スケール |
| `r.Lumen.DiffuseIndirect.MeshSDF.NotCoveredMinStepScale` | 32 | 両面マテリアル SDF の最小ステップスケール |
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
