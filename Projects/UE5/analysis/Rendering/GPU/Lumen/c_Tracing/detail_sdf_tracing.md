# Lumen SDF トレーシング シェーダー詳細

- グループ: c - Tracing
- GPU 概要: [[01_lumen_gpu_overview]]
- CPU 詳細: [[c_lumen_tracing]]
- リファレンス: [[ref_sdf_tracing]]

---

## 概要

Screen Probe・Radiance Cache・Radiosity の各トレースパスが共通で使う  
**SDF（Signed Distance Field）ベースのレイトレース** 基盤。

独立した GPU パスというよりは **共有ロジック＋カリングパス** であり、  
実際のトレースは各グループのシェーダーから `LumenSoftwareRayTracing.ush` / `LumenTracingCommon.ush` を `#include` して実行される。

---

## 2種類の SDF

| SDF | ファイル | 解像度 | 用途 |
|-----|---------|--------|------|
| **Mesh SDF** | メッシュごとに生成 | 高精度（近距離） | 近傍オブジェクトの詳細なトレース |
| **Global SDF** | シーン全体 | 低精度（遠距離 / Voxel ） | 遠距離・Mesh SDF 外のトレース |

---

## Mesh SDF カリング（LumenMeshSDFCulling.usf）

トレース前に各ビュー・各プローブに影響する Mesh SDF をカリングするパス。

```
[1] CullMeshSDFObjectsForViewCS()
      → View frustum × 距離カリング
      → VisibleMeshSDFsBuffer を生成

[2] CompactCulledObjectsCS()
      → 有効なエントリのみに Compact（連続配列化）

[3] ComputeCulledObjectsStartOffsetCS()
      → Tile/Probe ごとの開始オフセットを計算
```

### 主要 CS エントリポイント

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void CullMeshSDFObjectsForViewCS(uint3 DispatchThreadId : SV_DispatchThreadID)
// → 入力: MeshSDFObjectBounds[] / ViewFrustumPlanes
// → 出力: RWVisibleMeshSDFsBuffer（カリング後の SDF インデックス列）
```

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void CompactCulledObjectsCS(uint3 DispatchThreadId : SV_DispatchThreadID)
// → 出力: RWCompactedVisibleMeshSDFsBuffer
```

---

## Lumen Scene 更新（LumenScene.usf）

SDF 以外に、Lumen Scene データ（カードメタ情報）の GPU 側更新も担う。

```hlsl
[numthreads(THREADGROUP_SIZE, 1, 1)]
void LumenSceneUpdateCS(uint3 DispatchThreadId : SV_DispatchThreadID)
// → LumenCardData / LumenCardPageData の GPU バッファを CPU アップロードデータで更新
```

---

## トレース共通ロジック（ヘッダファイル群）

実際のレイ進行ロジックはヘッダとして提供され、各シェーダーが include する。

| ヘッダ | 役割 |
|-------|------|
| `LumenSoftwareRayTracing.ush` | SDF ステップマーチングのコアロジック（`TraceMeshSDF()` / `TraceGlobalSDF()`）|
| `LumenTracingCommon.ush` | トレース結果構造体（`FLumenMinHitResult`）・ヒット判定・FinalLighting サンプル |
| `LumenScreenTracing.ush` | HZB を使ったスクリーンスペーストレース（SST / Screen Trace）|
| `LumenHardwareRayTracingCommon.ush` | HW RT 共通ペイロード定義・マテリアル評価インターフェース |

### FLumenMinHitResult（トレース結果）

```hlsl
struct FLumenMinHitResult
{
    bool  bHit;                  // ヒットしたか
    float HitDistance;           // ヒット距離
    float3 HitNormal;            // ヒット法線
    uint  CardIndex;             // ヒットした Card インデックス
    float2 CardUV;               // Card 上の UV
};
```

---

## CPU 呼び出しの流れ

```
各トレースパス（Radiosity / ScreenProbe / RadianceCache）
  │
  ├─ CullMeshSDFsToProbes() または CullMeshSDFsToView()
  │   → LumenMeshSDFCulling.usf#CullMeshSDFObjectsForViewCS
  │
  └─ TraceMeshSDFs() / TraceGlobalSDF()（ヘッダ inline）
      → LumenSoftwareRayTracing.ush#TraceStepsMarchingToHit()
```

**パラメーター割り当て（概略）：**

```cpp
// 各トレース CS パラメーター構造体に共通で含まれる
FLumenMeshSDFGridParameters MeshSDFGridParameters;
MeshSDFGridParameters.NumMeshSDFs = NumVisibleMeshSDFs;
MeshSDFGridParameters.MeshSDFObjectBounds = GraphBuilder.CreateSRV(MeshSDF_BoundsBuffer);
MeshSDFGridParameters.MeshSDFObjects = GraphBuilder.CreateSRV(MeshSDF_DataBuffer);
// シェーダーに SHADER_PARAMETER_STRUCT_INCLUDE で伝搬
```
