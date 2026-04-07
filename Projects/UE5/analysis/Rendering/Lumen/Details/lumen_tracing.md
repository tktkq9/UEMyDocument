# Lumen Tracing（ライトトレース）

- 上位: [[02_lumen_overview]]
- 関連: [[lumen_surface_cache]] | [[lumen_diffuse_gi]] | [[lumen_reflections]]

---

## 概要

「あるピクセル（または Probe）からある方向にレイを飛ばすと何が見えるか」を解決する層。  
Lumen は **3種類のトレース方法** を持ち、距離・設定・ハードウェア対応に応じて使い分ける。

ヒットした地点のライティング情報は **[[lumen_surface_cache]] の FinalLightingAtlas** から取得する。

---

## トレース方法の比較

| 方法 | 距離 | 精度 | GPU コスト | 主なファイル |
|------|------|------|-----------|------------|
| **Mesh SDF** | 近距離（〜180cm） | 高（メッシュ形状に追従） | 中 | `LumenMeshSDFCulling.cpp` |
| **Global SDF（Voxel）** | 中〜遠距離 | 中（ボクセル近似） | 低 | `DistanceFieldAmbientOcclusion.cpp` |
| **HW Ray Tracing** | 全距離 | 最高 | 高 | `LumenHardwareRayTracingCommon.cpp` |

---

## ETracingPermutation（組み合わせパターン）

```cpp
// Lumen.h 内で定義
enum class ETracingPermutation {
    Cards,              // Surface Cache のみ（最速・近距離の面へのヒット）
    VoxelsAfterCards,   // Cards → ミスしたら Global SDF にフォールバック
    Voxels,             // Global SDF のみ
};
```

実際には距離に応じて自動選択される：
```
近距離: MeshSDF → Cards（Surface Cache ヒット）
中距離: GlobalSDF → VoxelsAfterCards
遠距離: Radiance Cache からサンプリング（→ [[lumen_radiance_cache]]）
```

---

## 1. Mesh SDF Tracing（近距離）

### SDF とは

Signed Distance Field（符号付き距離場）：  
各ボクセルに「最も近いメッシュ表面までの距離」を格納したテクスチャ。  
これを使うと「今いる点からメッシュ表面まで安全に進める距離」が O(1) で分かる。

```
通常のレイマーチ: 毎ステップ固定刻み → 表面を通り抜ける危険
SDF レイマーチ:  SDF値分だけ進む → 絶対に表面を見落とさない（Sphere Tracing）
```

### FLumenMeshSDFTracingParameters

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenMeshSDFTracingParameters, )
    // UE の Distance Field システムとの共有バッファ
    SHADER_PARAMETER_STRUCT_INCLUDE(FDistanceFieldObjectBufferParameters, DistanceFieldObjectBuffers)
    SHADER_PARAMETER_STRUCT_INCLUDE(FDistanceFieldAtlasParameters, DistanceFieldAtlas)

    float MeshSDFNotCoveredExpandSurfaceScale;   // カバーされない部分の拡張係数
    float MeshSDFNotCoveredMinStepScale;          // 最小ステップスケール
    float MeshSDFDitheredTransparencyStepThreshold; // 半透明ディザのしきい値
END_SHADER_PARAMETER_STRUCT()
```

### FLumenMeshSDFGridParameters（グリッドカリング）

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FLumenMeshSDFGridParameters, )
    SHADER_PARAMETER_STRUCT_INCLUDE(FLumenMeshSDFTracingParameters, TracingParameters)

    // ビュー空間をグリッドに分割し、各セルに影響するSDFオブジェクトを列挙
    Buffer<uint> NumGridCulledMeshSDFObjects        // セルあたりのSDFオブジェクト数
    Buffer<uint> GridCulledMeshSDFObjectStartOffsetArray // 各セルの開始オフセット
    Buffer<uint> GridCulledMeshSDFObjectIndicesArray    // オブジェクトインデックス

    uint32 CardGridPixelSizeShift;  // グリッドセルのピクセルサイズ（2のべき乗）
    FVector3f CardGridZParams;      // 深度方向のグリッドパラメータ
    FIntVector CullGridSize;        // グリッド全体の解像度
END_SHADER_PARAMETER_STRUCT()
```

### デフォルトトレース距離

```cpp
// FLumenGatherCvarState の初期値（LumenDiffuseIndirect.cpp）
MeshSDFTraceDistance = 180.0f;  // 約1.8m（UE 単位: cm）
```

---

## 2. Global SDF Tracing（中〜遠距離）

### Global Distance Field とは

シーン全体をカバーする低解像度のボクセル SDF。  
クリップマップ方式（カメラ中心に複数段の解像度）で管理される。

```
Mesh SDF: 個々のメッシュに対して高解像度（精度高・範囲狭）
Global SDF: シーン全体を低解像度でカバー（精度中・範囲広）
```

### クリップマップ定数（Lumen.h より）

```cpp
constexpr uint32 NumDistanceBuckets = 16;  // 距離バケット数
// 最大クリップマップ段数（LumenViewState.h より）
const static int32 MaxVoxelClipmapLevels = 8;
```

### Voxel トレース設定

```
r.Lumen.DiffuseIndirect.VoxelTracingMode = 0
    0: Cards が届かない場所のみ Voxel を使う
    1: 全て Voxel で処理
    2: Cards のみ（Voxel 無効）
```

---

## 3. HW Ray Tracing（オプション・高品質）

### 概要

DXR（DirectX Raytracing）または Vulkan RT を使った  
GPU ハードウェアによる真のレイトレーシング。

```cpp
// SceneRendering.h より（条件コンパイル）
#if RHI_RAYTRACING
    #include "RayTracing/RayTracingScene.h"
#endif
```

### Nanite との関係

Nanite メッシュは通常の Ray Tracing には直接使えない（BVH が異なる）。  
Lumen の HW RT 用に別途 **Fallback Mesh** を生成して BVH に登録する。

```
NaniteRayTracing.cpp → Nanite メッシュの HW RT 対応
LumenHardwareRayTracingCommon.cpp → Lumen 用 HW RT の共通処理
LumenHardwareRayTracingMaterials.cpp → マテリアルヒット処理
```

---

## Surface Cache ヒット後の処理

レイがヒットした後、ヒット地点のライティングを取得する共通処理。

### FLumenCardTracingParameters（全トレーサーが共有）

```cpp
// LumenTracingUtils.h より（抜粋）
BEGIN_SHADER_PARAMETER_STRUCT(FLumenCardTracingParameters, )
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FLumenCardScene, LumenCardScene)

    // Surface Cache の各アトラス（読み取り専用）
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DirectLightingAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, IndirectLightingAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, FinalLightingAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, AlbedoAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, NormalAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, EmissiveAtlas)
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, DepthAtlas)

    // フィードバックバッファ（どのCardが参照されたかを記録）
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>, RWCardPageLastUsedBuffer)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint2>, RWSurfaceCacheFeedbackBuffer)

    // ライティング調整
    float DiffuseColorBoost;
    FVector3f SkylightLeakingColor;
    float CachedLightingPreExposure;
END_SHADER_PARAMETER_STRUCT()
```

### ヒット時の輝度サンプリング

```
レイがヒット（Card に命中）
  → ヒット地点の Card ページを特定
  → FinalLightingAtlas の対応テクセルをサンプリング
  → 法線・距離に応じた重みで輝度を取得
  → RWSurfaceCacheFeedbackBuffer に参照記録（→ [[lumen_surface_cache]] の Feedback）
```

---

## FHemisphereDirectionSampleGenerator（サンプル方向生成）

```cpp
// 半球方向のサンプル生成ユーティリティ（LumenTracingUtils.h）
class FHemisphereDirectionSampleGenerator {
    TArray<FVector4f> SampleDirections;  // 事前生成したサンプル方向リスト
    float ConeHalfAngle;
    bool bFullSphere;           // 完全球（両面）か半球か
    bool bCosineDistribution;   // コサイン重み付き分布か

    void GenerateSamples(int32 TargetNumSamples, int32 PowerOfTwoDivisor,
                         int32 Seed, bool bFullSphere, bool bCosineDistribution);
};
```

---

## 主要 CVar

```
# SDF トレース
r.Lumen.DiffuseIndirect.TraceMeshSDFs = 1
r.Lumen.DiffuseIndirect.MeshSDFTraceDistance = 180.0
r.Lumen.DiffuseIndirect.SurfaceBias = 5.0       ← 自己交差防止バイアス
r.Lumen.DiffuseIndirect.VoxelTracingMode = 0

# HW Ray Tracing
r.Lumen.HardwareRayTracing = 0     ← 0:Software  1:HW RT
r.Lumen.HardwareRayTracing.LightingMode = 0

# Global SDF
r.Lumen.GlobalSDF.Resolution
```

---

## 関連ソースファイル

| ファイル | 役割 |
|---------|------|
| `LumenTracingUtils.h/cpp` | 共通トレース構造体・ユーティリティ |
| `LumenMeshSDFCulling.cpp` | Mesh SDF のグリッドカリング |
| `LumenHardwareRayTracingCommon.h/cpp` | HW RT 共通処理 |
| `LumenHardwareRayTracingMaterials.cpp` | HW RT マテリアルヒット処理 |
| `NaniteRayTracing.h/cpp` | Nanite の HW RT 対応 |
| `GlobalDistanceField.cpp` | Global SDF の更新管理 |
| `DistanceFieldAmbientOcclusion.cpp` | Global SDF トレース |
