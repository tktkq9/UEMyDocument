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

---

## FLumenHeightfieldGPUData

Heightfield 1 枚分の GPU バッファデータレイアウト定義。  
シェーダー側の `GetLumenHeightfieldData()` と一致するレイアウトで FVector4f 配列に書き込む。

```cpp
struct FLumenHeightfieldGPUData {
    // LumenCardCommon.ush の LUMEN_HEIGHTFIELD_DATA_STRIDE と一致
    enum { DataStrideInFloat4s = 3 };              // 1 Heightfield = 3 × float4
    enum { DataStrideInBytes = 3 * 16 };           // 48 bytes

    // FVector4f[3] に以下の順でデータを書き込む:
    //   [0]     : BoundsCenter.High（DF64 精度, W = MeshCardsIndex（float bit cast））
    //   [1]     : BoundsCenter.Low（DF64 低精度）
    //   [2]     : BoundsExtent（ワールド空間 AABB の半サイズ）
    static void FillData(
        const FLumenHeightfield& RESTRICT Heightfield,
        const TSparseSpanArray<FLumenMeshCards>& MeshCards,
        FVector4f* RESTRICT OutData);
};
```

---

## Lumen 名前空間（Heightfield 判定関数）

```cpp
namespace Lumen {
    // Voxel Lighting に Heightfield トレースを使うか
    // 条件: CVar 有効 && シーンに Heightfield が存在
    bool UseHeightfieldTracingForVoxelLighting(const FLumenSceneData& LumenSceneData);

    // Detail トレース（Screen Probe 等）に Heightfield を使うか
    // 条件: UseHeightfieldTracingForVoxelLighting() && UseMeshSDFTracing() && LumenDetailTraces
    bool UseHeightfieldTracing(const FSceneViewFamily& ViewFamily, const FLumenSceneData& LumenSceneData);

    // Heightfield の最大トレースステップ数（1〜256 にクランプ）
    int32 GetHeightfieldMaxTracingSteps();

    // Heightfield レシーバーバイアス（Landscape LOD ミスマッチ補正）
    float GetHeightfieldReceiverBias();
}
```

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

## FillData のデータレイアウト（シェーダー対応）

```
FVector4f[0]: { BoundsCenter.High.X, BoundsCenter.High.Y, BoundsCenter.High.Z, *(float*)&MeshCardsIndex }
FVector4f[1]: { BoundsCenter.Low.X,  BoundsCenter.Low.Y,  BoundsCenter.Low.Z,  0 }
FVector4f[2]: { BoundsExtent.X,      BoundsExtent.Y,      BoundsExtent.Z,      0 }

// MeshCardsIndex は uint32 を float にビットキャストして W 成分に格納
// シェーダー側: asuint(Data[0].w) で復元
```

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

[HW RT 使用時]
  → Landscape は RT BLAS を持つため、HW RT でも正確なトレースが可能
  → LumenSceneDirectLightingHardwareRayTracing.cpp で HeightfieldProjectionBias を使用
```
