# リファレンス：LumenHeightfields.h / LumenHeightfields.cpp

- グループ: c - Tracing
- 上位: [[c_lumen_tracing]]
- 関連: [[ref_lumen_tracing_utils]] | [[ref_lumen_mesh_sdf_culling]] | [[ref_lumen_mesh_cards]]
- ソース:
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenHeightfields.h`
  - `Engine/Source/Runtime/Renderer/Private/Lumen/LumenHeightfields.cpp`

---

## 概要

Lumen における **Landscape（Heightfield）** のトレーシングサポート。  
Heightfield は Mesh SDF ではなく専用のハイトマップトレース（レイマーチ）で  
ソフトウェアレイトレーシングされる。GPU データレイアウトを定義し、  
Lumen の SDF トレーシングパイプラインに組み込む。

---

## FLumenHeightfield

Lumen シーン内の個々の Heightfield（Landscape コンポーネント）を表すクラス。  
`FLumenMeshCards` と 1:1 で対応し、`MeshCardsIndex` で参照する。

```cpp
class FLumenHeightfield {
public:
    int32 MeshCardsIndex = -1; // FLumenSceneData::MeshCards 内のインデックス（-1=無効）

    void Initialize(int32 InMeshCardsIndex);
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `MeshCardsIndex` | `int32` | `FLumenSceneData::MeshCards` 配列内のインデックス。-1 は無効（未初期化）|

### 使用箇所

- [[ref_lumen_scene_data]] — `FLumenSceneData::Heightfields` 配列に格納される
- [[ref_lumen_heightfields]] — `FLumenHeightfieldGPUData::FillData()` の引数として渡される
- [[ref_lumen_mesh_sdf_culling]] — `CullHeightfieldObjectsForView()` でビューカリング対象

---

## FLumenHeightfieldGPUData

Heightfield 1 枚分の GPU バッファデータレイアウト定義。  
シェーダー側の `GetLumenHeightfieldData()` と一致するレイアウトで FVector4f 配列に書き込む。

```cpp
struct FLumenHeightfieldGPUData {
    enum { DataStrideInFloat4s = 3 };    // 1 Heightfield = 3 × float4
    enum { DataStrideInBytes = 3 * 16 }; // 48 bytes

    static void FillData(
        const FLumenHeightfield& RESTRICT Heightfield,
        const TSparseSpanArray<FLumenMeshCards>& MeshCards,
        FVector4f* RESTRICT OutData);
};
```

### 定数

| 定数 | 値 | 説明 |
|------|-----|------|
| `DataStrideInFloat4s` | 3 | 1 Heightfield あたりの float4 要素数。シェーダーの `LUMEN_HEIGHTFIELD_DATA_STRIDE` と一致 |
| `DataStrideInBytes` | 48 | バイト換算（3 × 16 bytes）|

### FillData の内部処理フロー

```cpp
static void FillData(
    const FLumenHeightfield& RESTRICT Heightfield,
    const TSparseSpanArray<FLumenMeshCards>& MeshCards,
    FVector4f* RESTRICT OutData);
```

**パラメータ**

| 引数 | 型 | 説明 |
|------|-----|------|
| `Heightfield` | `const FLumenHeightfield&` | GPU データを生成する Heightfield |
| `MeshCards` | `const TSparseSpanArray<FLumenMeshCards>&` | MeshCards 配列（AABB 情報の取得用）|
| `OutData` | `FVector4f*` | 出力先バッファポインタ（3 × float4 分の書き込み）|

1. **MeshCards から AABB を取得**
   ```cpp
   const FLumenMeshCards& LumenMeshCards = MeshCards[Heightfield.MeshCardsIndex];
   const FBox WorldBounds = LumenMeshCards.GetWorldSpaceBounds();
   const FVector BoundsCenter = WorldBounds.GetCenter();
   const FVector BoundsExtent = WorldBounds.GetExtent();
   ```

2. **DF64 精度での BoundsCenter の分割（浮動小数精度対策）**
   ```cpp
   // DF64: 倍精度相当の精度を2つのfloatで実現
   FVector BoundsCenterHigh = ...; // 上位ビット（粗い値）
   FVector BoundsCenterLow  = ...; // 下位ビット（補正値）
   ```

3. **MeshCardsIndex のビットキャスト格納**
   ```cpp
   // W 成分に MeshCardsIndex を float ビットキャストして格納
   // シェーダー側: asuint(Data[0].w) で復元
   OutData[0] = FVector4f(BoundsCenterHigh.X, BoundsCenterHigh.Y, BoundsCenterHigh.Z,
                           *(float*)&Heightfield.MeshCardsIndex);
   OutData[1] = FVector4f(BoundsCenterLow.X,  BoundsCenterLow.Y,  BoundsCenterLow.Z, 0);
   OutData[2] = FVector4f(BoundsExtent.X,     BoundsExtent.Y,     BoundsExtent.Z,    0);
   ```

### 使用箇所

- [[ref_lumen_scene_data]] — `UpdateLumenSceneBuffers()` で Heightfield GPU バッファ更新時に呼ばれる

---

## FillData のデータレイアウト（シェーダー対応）

```
FVector4f[0]: { BoundsCenter.High.X, BoundsCenter.High.Y, BoundsCenter.High.Z, *(float*)&MeshCardsIndex }
FVector4f[1]: { BoundsCenter.Low.X,  BoundsCenter.Low.Y,  BoundsCenter.Low.Z,  0 }
FVector4f[2]: { BoundsExtent.X,      BoundsExtent.Y,      BoundsExtent.Z,      0 }

// MeshCardsIndex は uint32 を float にビットキャストして W 成分に格納
// シェーダー側: asuint(Data[0].w) で復元
```

---

## Lumen 名前空間（Heightfield 判定関数）

```cpp
namespace Lumen {
    bool UseHeightfieldTracingForVoxelLighting(const FLumenSceneData& LumenSceneData);
    bool UseHeightfieldTracing(const FSceneViewFamily& ViewFamily, const FLumenSceneData& LumenSceneData);
    int32 GetHeightfieldMaxTracingSteps();
    float GetHeightfieldReceiverBias();
}
```

| 関数 | 戻り値 | 説明 |
|------|--------|------|
| `UseHeightfieldTracingForVoxelLighting()` | `bool` | Voxel Lighting に Heightfield トレースを使うか。条件: CVar 有効 && シーンに Heightfield が存在 |
| `UseHeightfieldTracing()` | `bool` | Detail トレース（Screen Probe 等）に Heightfield を使うか。追加条件: UseMeshSDFTracing() が有効 |
| `GetHeightfieldMaxTracingSteps()` | `int32` | CVar `r.LumenScene.Heightfield.MaxTracingSteps` を 1〜256 にクランプして返す |
| `GetHeightfieldReceiverBias()` | `float` | CVar `r.LumenScene.Heightfield.ReceiverBias` を返す（Landscape LOD ミスマッチ補正）|

### 使用箇所

- [[ref_lumen_tracing_utils]] — `FLumenIndirectTracingParameters::HeightfieldMaxTracingSteps` の初期化に `GetHeightfieldMaxTracingSteps()` を使用
- [[ref_lumen_mesh_sdf_culling]] — `CullForCardTracing()` 内で `UseHeightfieldTracing()` を確認してカリング分岐
- [[ref_lumen_screen_probe_tracing]] — Screen Probe トレースパスで `UseHeightfieldTracing()` を判定

> [!note]- UseHeightfieldTracingForVoxelLighting vs UseHeightfieldTracing の違い
>
> ```
> UseHeightfieldTracingForVoxelLighting():
>   → Voxel Lighting（Global SDF / Radiance Cache）パスで Heightfield を使うかどうか
>   → 条件: r.LumenScene.Heightfield.Tracing = 1 かつ Scene に Heightfield が 1 枚以上存在
>
> UseHeightfieldTracing():
>   → Screen Probe・Detail Trace など高精度パスで Heightfield を使うかどうか
>   → 追加条件: UseMeshSDFTracing() が有効（SDF トレースが有効な場合のみ Heightfield も有効）
>   → 一般的に UseHeightfieldTracingForVoxelLighting() の部分集合
> ```

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|-----------|------|
| `r.LumenScene.Heightfield.Tracing` | 1 | Heightfield の SW RT 有効/無効 |
| `r.LumenScene.Heightfield.MaxTracingSteps` | 32 | ハイトマップレイマーチの最大ステップ数 |
| `r.LumenScene.Heightfield.ReceiverBias` | 0.01 | 受信側バイアス（Landscape の固定 LOD とランタイム CLOD のずれ補正）|
| `r.LumenScene.Heightfield.CullForView` | 1 | ビューカリングの有効/無効 |
| `r.LumenScene.Heightfield.FroxelCulling` | 1 | フロクセルカリングの有効/無効 |

---

## Heightfield トレーシングの概要

```
[CPU] FLumenSceneData::Heightfields に FLumenHeightfield を追加
  → FillData() で GPU バッファを生成・アップロード
  → CullHeightfieldObjectsForView() でビューカリング

[GPU] Heightfield トレース（レイマーチ方式）
  → ハイトマップテクスチャに対してレイマーチ
  → MaxTracingSteps 回ステップしてヒット判定
  → Mesh SDF と異なり、距離フィールドを持たないため精度に限界あり
  → ステップ数が少ないと薄い Landscape 地形を貫通することがある

[HW RT 使用時]
  → Landscape は RT BLAS を持つため、HW RT でも正確なトレースが可能
  → LumenSceneDirectLightingHardwareRayTracing.cpp で HeightfieldProjectionBias を使用
```
