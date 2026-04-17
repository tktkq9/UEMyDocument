# GPU b: Global SDF — クリップマップ合成・Mip 生成シェーダー

- シェーダー: `DistanceField/GlobalDistanceField.usf`, `GlobalDistanceFieldCompositeObjects.usf`, `GlobalDistanceFieldMip.usf`
- CPU 対応: [[b_global_sdf]] | [[ref_global_sdf]]
- 上位: [[01_distance_field_gpu_overview]]

---

## 概要

Global SDF はシーン全体のワールド SDF をクリップマップ形式で管理する。  
GPU パスでは **Mesh SDF をクリップマップボクセルへ合成**（ComposeObjects）し、  
続いて **粗い Mip テクスチャを生成**（GenerateMip）する。  
いずれも Compute Shader で動作し、差分更新（カメラ追従スライディング）をサポートする。

---

## シェーダーファイル構成

| ファイル | 主要 CS エントリ | 役割 |
|---------|---------------|------|
| `GlobalDistanceFieldCompositeObjects.usf` | `ComposeObjectsIntoPagesCS` | Mesh SDF → クリップマップページへ合成 |
| `GlobalDistanceField.usf` | `InitializePageFreeListCS`<br>`AllocatePagesCS`<br>`PageClearCS` | ページ管理・初期化 |
| `GlobalDistanceFieldMip.usf` | `GenerateMipCS` | クリップマップから低解像度 Mip 生成 |
| `GlobalDistanceFieldHeightfields.usf` | `ComposeHeightfieldsIntoClipmap` | 地形 SDF の合成 |

---

## ページベース構造（Sparse Global SDF）

UE5 の Global SDF は**ページベース（スパース）**で管理される。

```
クリップマップ（4–8 段）
  │
  ├─ PageTableTexture（Texture3D<uint>）
  │   各ボクセルが「どのページに格納されているか」を示すインデックス
  │
  └─ PageAtlasTexture（Texture3D）
      実際の SDF 値が格納される 3D テクスチャ（ページ集合体）
      未使用ページはアトラスに割り当てられない（スパース）

CoverageAtlasTexture（Texture3D）
  → オブジェクトがカバーしているか否かの情報（DFAO の精度向上用）
```

---

## ComposeObjectsIntoPagesCS

`GlobalDistanceFieldCompositeObjects.usf`

Mesh SDF から Global SDF ページへ距離値を書き込む中核シェーダー。

```hlsl
// スレッド1つが1ページ内の1ボクセルを担当
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, THREADGROUP_SIZE)]
void ComposeObjectsIntoPagesCS(uint3 GroupId : SV_GroupID, ...)
{
    // 1. VoxelWorldCenter を計算（クリップマップ座標 → ワールド座標）
    FDFVector3 VoxelWorldCenter = ...;

    // 2. このボクセルに影響する Mesh SDF オブジェクトをカリング
    //    CullGrid から対象オブジェクトリストを取得

    // 3. 各 Mesh SDF から距離値を取得・最小値を保持
    float MinDistance = InfluenceRadius;
    for each ObjectIndex in CulledObjects:
        CompositeMeshSDF(ObjectIndex, VoxelWorldCenter, MinDistance, ...);

    // 4. PageAtlasTexture へ書き込み
    RWPageAtlasTexture[PageVoxelPos] = saturate(MinDistance / InfluenceRadius);
}
```

**CompositeMeshSDF の処理：**

```hlsl
void CompositeMeshSDF(uint ObjectIndex, FDFVector3 VoxelWorldCenter,
                      inout float MinDistance, ...)
{
    FDFObjectData DFObjectData = LoadDFObjectData(ObjectIndex);
    float DistanceToOccluder = DistanceToNearestSurfaceForObject(
        DFObjectData, VoxelWorldCenter, MaxQueryDistance);
    MinDistance = min(MinDistance, DistanceToOccluder);
}
```

---

## カリンググリッド（CullGrid）

効率のため、ボクセル空間を格子（CullGrid）に分割し、各格子セルに影響する Mesh SDF オブジェクトのリストをあらかじめ構築する。

```hlsl
StructuredBuffer<uint> CullGridObjectHeader;  // 格子セルごとのリスト先頭インデックス
StructuredBuffer<uint> CullGridObjectArray;   // オブジェクトインデックスリスト
uint3 CullGridResolution;                     // 格子解像度
```

- `USE_CULL_GRID 1` が定義されている場合に有効（デフォルト有効）

---

## GenerateMipCS

`GlobalDistanceFieldMip.usf`

PageAtlasTexture から低解像度 MipTexture を生成する。

```hlsl
[numthreads(THREADGROUP_SIZE, THREADGROUP_SIZE, THREADGROUP_SIZE)]
void GenerateMipCS(uint3 DispatchThreadId : SV_DispatchThreadID)
{
    // 2x2x2 の上位解像度 SDF を読み取り
    // → MipFactor によるスケールで統合
    // → 粗い Mip テクスチャへ書き込み
}
```

Mip テクスチャは遠距離でのサンプリングコスト削減のために使用される。

---

## 差分更新（スライディング）アルゴリズム

```
カメラ移動量を計算
  │
  ├─ 小移動（1クリップマップ以内）
  │   → UpdateBounds（差分領域）のみページを再合成
  │   → 移動していない領域はキャッシュ済みのページを再利用
  │
  └─ 大移動 / カメラカット
      → 全クリップマップのページを無効化
      → 全面再合成（AllocatePagesCS + ComposeObjectsIntoPagesCS）
```

---

## GDF_MostlyStatic vs GDF_Full

| キャッシュ種別 | 合成対象 | 更新タイミング |
|-------------|---------|--------------|
| `GDF_MostlyStatic` | 静的 Mesh SDF のみ | 静的オブジェクト変化時 or カメラ移動 |
| `GDF_Full` | 静的 + 動的 Mesh SDF | 毎フレーム（動的オブジェクトが含まれる領域）|

GDF_Full は GDF_MostlyStatic をベースにして動的オブジェクトを上書き合成する。

---

## 地形 SDF（HeightfieldSDF）

`GlobalDistanceFieldHeightfields.usf`

- ランドスケープのハイトマップから SDF を生成してクリップマップへ合成する
- `Landscape.SDF` アセットが有効な場合のみ動作

---

## バインドパラメータ

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `RWPageAtlasTexture` | `RWTexture3D<UNORM float>` | 出力: SDF 値アトラス |
| `RWCoverageAtlasTexture` | `RWTexture3D<UNORM float>` | 出力: カバレッジ情報 |
| `PageTableLayerTexture` | `Texture3D<uint>` | ページテーブル（読み取り）|
| `RWPageTableCombinedTexture` | `RWTexture3D<uint>` | 出力: 統合ページテーブル |
| `InfluenceRadius` | `float` | オブジェクト影響半径 |
| `ClipmapVoxelExtent` | `float` | ボクセル1辺の実距離 |
| `CullGridResolution` | `uint3` | カリンググリッド解像度 |
