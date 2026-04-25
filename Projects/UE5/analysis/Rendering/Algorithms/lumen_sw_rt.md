# Lumen Software Ray Tracing（Mesh Distance Fields + Global SDF）

- 上位: [[_algorithm_index]]
- 関連: [[lumen_hw_rt]] / [[lumen_surface_cache]] / [[lumen_final_gather]]
- 採用システム: Lumen GI / Lumen Reflections の SW トレース経路（HW RT が無い・OFF のとき）, DistanceFieldShadows
- 出典:
  - **S10**: Wright / Heitz / Hillaire 2022 SIGGRAPH — §4 Software Ray Tracing
  - 影響元: Wright "Mesh Distance Field"（UE4 から継続）, Crassin "Global SDF" 系

---

## 1. 何のためのアルゴリズムか

GPU の Hardware Ray Tracing が使えない / 重すぎる環境（旧 GPU、コンソール、Forward Renderer 一部）で **「ほぼ同等品質の」レイトレースを compute シェーダで行う**。

### 素朴な手法の問題

- **Triangle BVH を compute で回す**: 性能が出ない、CPU からの BLAS 構築コスト
- **完全 Voxel grid**: 解像度を上げると VRAM 爆発
- **Cone Tracing 単独**: 細部欠損、leaking

### Lumen SW RT の貢献

- 各メッシュに **Mesh Distance Field (MDF, 8-bit narrow-band SDF)** を **オフライン** で生成
- 全シーンを **Global SDF (Sparse 3D Clipmap)** にラスタライズし、毎フレーム更新
- レイは **Sphere Tracing**（SDF の値だけ進む安全な距離を進む）でヒットを探す
- ヒット点は **Mesh Cards への変換** で [[lumen_surface_cache]] から放射輝度取得
- HW RT より **粗い**（SDF 解像度依存）が **十分に近い品質**

---

## 2. 理論

### 2.1 Sphere Tracing（Distance Field Ray Marching）

```
P = rayOrigin
while (t < maxDist) {
    d = SDF(P)
    if (d < epsilon) return HIT(P)   // ヒット
    P += rayDir * d
    t += d
}
return MISS
```

- **SDF の幾何的保証**: `SDF(P) = 表面までの最短距離` → そのぶん必ず安全に進める
- イテレーション数は SDF の凹凸に依存、典型 16〜64 ステップ
- 三角形交差より遅いがメモリアクセス局所性が高く、compute で並列化容易

### 2.2 階層構造

| レベル | 用途 | 解像度 |
|-------|------|-------|
| **Mesh Distance Field (MDF)** | 個別メッシュごと、AABB 近傍のみ narrow-band | メッシュサイズ依存（典型 64³） |
| **Global SDF** | シーン全体の Sparse 3D Clipmap、Mesh SDF を合成 | クリップマップ 4 段 |

### 2.3 Lumen のレイ階層

`r.Lumen.DiffuseIndirect.MeshSDF.*` 系の CVar から推察される 3 段戦略:

1. **Screen Trace（HZB）**: スクリーン空間で短距離（典型 0〜数十 cm）を高速ヒット判定
2. **Mesh SDF Trace**: スクリーンを抜けたレイは、view frustum 内のカリング済み Mesh SDF オブジェクトに対して個別 SDF トレース
3. **Global SDF Trace**: Mesh SDF が無い領域（カリング outside）は Global SDF でフォールバック

`Lumen::ETracingPermutation::{Cards, VoxelsAfterCards, Voxels}` (`Lumen.h:51-57`) が permutation 識別子。

### 2.4 Mesh SDF からカードへの変換

SDF はジオメトリ情報のみ（material 無し）。ヒット点 → `MeshCardsIndex` を引き、最も法線に合うカードに UV 投影 → Surface Cache サンプリング。

詳細は [[lumen_surface_cache]] §2.1。

### 2.5 Coverage Expand

低解像度 SDF はメッシュの「実体」より少し膨らんで見える。これを補正するために `MeshSDFNotCoveredExpandSurfaceScale = 0.6` で補正係数を掛ける（特に Foliage の two-sided マテリアル）。

---

## 3. UE での実装

### 3.1 主要ファイル

| ファイル | 役割 |
|---------|------|
| `LumenMeshSDFCulling.cpp` | View frustum 内の Mesh SDF オブジェクトをカリング（compute） |
| `LumenScreenProbeTracing.cpp` | ScreenProbe レイの SW トレース経路 |
| `LumenReflectionTracing.cpp` | Reflection レイの SW トレース経路 |
| `LumenTracingUtils.cpp` | 共通 SDF サンプリング |
| `DistanceFieldLightingShared.h` | Mesh SDF / Global SDF アトラスバインディング |
| `Lumen*.usf` (shader) | Sphere Tracing 本体 |

### 3.2 カリング compute（`LumenMeshSDFCulling.cpp:88-121`）

```cpp
class FCullMeshSDFObjectsForViewCS : public FGlobalShader
{
    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
        SHADER_PARAMETER_RDG_BUFFER_UAV(RWBuffer<uint>, RWNumCulledObjects)
        SHADER_PARAMETER_RDG_BUFFER_UAV(RWBuffer<uint>, RWObjectIndexBuffer)
        SHADER_PARAMETER_STRUCT_INCLUDE(FDistanceFieldObjectBufferParameters, DistanceFieldObjectBuffers)
        SHADER_PARAMETER_ARRAY(FVector4f, ViewFrustumConvexHull, [6])
        SHADER_PARAMETER(float, CardTraceEndDistanceFromCamera)
        SHADER_PARAMETER(float, MaxMeshSDFInfluenceRadius)
        SHADER_PARAMETER(float, MeshSDFRadiusThreshold)
    END_SHADER_PARAMETER_STRUCT()

    static constexpr uint32 GroupSize = 64;
};
```

ビューフラスタム 6 平面で Mesh SDF AABB をカリングし、`ObjectIndexBuffer` に通過オブジェクトを書く。

### 3.3 Tracing パラメータ（`LumenMeshSDFCulling.cpp:72-84`）

```cpp
void SetupLumenMeshSDFTracingParameters(
    FRDGBuilder& GraphBuilder, const FScene* Scene, const FViewInfo& View,
    FLumenMeshSDFTracingParameters& OutParameters)
{
    OutParameters.DistanceFieldObjectBuffers = DistanceField::SetupObjectBufferParameters(...);
    OutParameters.DistanceFieldAtlas = DistanceField::SetupAtlasParameters(...);
    OutParameters.MeshSDFNotCoveredExpandSurfaceScale = GMeshSDFNotCoveredExpandSurfaceScale;  // 0.6
    OutParameters.MeshSDFNotCoveredMinStepScale = GMeshSDFNotCoveredMinStepScale;              // 32
    OutParameters.MeshSDFDitheredTransparencyStepThreshold = GMeshSDFDitheredTransparencyStepThreshold;  // 0.1
}
```

### 3.4 主要 CVar

| CVar | デフォルト | 効果 |
|------|----------|------|
| `r.Lumen.DiffuseIndirect.MeshSDF.AverageCulledCount` | 512 | カリング後オブジェクト数の予算（メモリ配分） |
| `r.Lumen.DiffuseIndirect.MeshSDF.RadiusThreshold` | 30 cm | この半径未満の Mesh SDF はトレース対象外 |
| `r.Lumen.DiffuseIndirect.MeshSDF.NotCoveredExpandSurfaceScale` | 0.6 | Foliage 用補正 |
| `r.Lumen.DiffuseIndirect.MeshSDF.NotCoveredMinStepScale` | 32 | Foliage 用最小ステップスケール |
| `r.LumenScene.Heightfield.CullForView` | 1 | Heightfield（Landscape）も SDF と並行カリング |
| `r.LumenScene.Heightfield.FroxelCulling` | 1 | Froxel 単位の Heightfield カリング |

### 3.5 グローバル SDF 関連 API（`Lumen.h:71-96`）

```cpp
namespace Lumen {
    int32 GetGlobalDFResolution();
    float GetGlobalDFClipmapExtent(int32 ClipmapIndex);
    int32 GetNumGlobalDFClipmaps(const FSceneView& View);

    bool UseMeshSDFTracing(const FEngineShowFlags& EngineShowFlags);
    bool UseGlobalSDFTracing(const FEngineShowFlags& EngineShowFlags);
    bool UseGlobalSDFSimpleCoverageBasedExpand();
    bool UseGlobalSDFObjectGrid(const FSceneViewFamily& ViewFamily);
    bool UseHeightfieldTracing(const FSceneViewFamily& ViewFamily, const FLumenSceneData& LumenSceneData);
    int32 GetHeightfieldMaxTracingSteps();
    bool IsUsingGlobalSDF(const FSceneViewFamily& ViewFamily);
}
```

---

## 4. 近似・省略の差分

| 項目 | HW RT（参考） | Lumen SW RT | 影響 |
|------|------------|------------|------|
| 表現 | Triangle BVH（ピクセル精度） | SDF（narrow-band） | 細部 detail loss、薄い壁で leak |
| 解像度 | 完全 | Mesh SDF ~64³, Global SDF Clipmap | 細い枝葉・配管で MISS |
| ステップ数 | BVH traverse | Sphere Trace 16〜64 step | 凹凸激しい SDF で多ステップ |
| 透過 | Any-Hit Shader | Dithered Transparency 単純閾値 | 半透明品質低下 |
| 動的 BLAS | 必要 | Global SDF 再ラスタライズで近似 | 完全動的形状は遅延 |
| Shadow | レイトレース | コーン状 SDF サンプリング | ソフトシャドウは標準 |

---

## 5. パラメータと CVar

§3.4 にまとめ済み。`r.Lumen.HardwareRayTracing = 0` で SW RT モードに強制可能。

---

## 6. 代替手法との比較

| 手法 | 表現 | コスト | UE 採用 |
|------|------|------|--------|
| Triangle BVH（HW RT） | 三角形 | RTX 系で軽い、それ以外は重い | [[lumen_hw_rt]] |
| **Mesh SDF + Global SDF** | **narrow-band SDF + sparse 3D** | **compute で一定コスト** | **Lumen SW RT 標準** |
| Voxel Cone Tracing | 完全 voxel pyramid | VRAM 重 | 未採用（VXGI 系） |
| Sparse Voxel Octree | SVO | 構築コスト高 | 未採用 |
| Screen Space Tracing 単独 | HZB | 軽 | Lumen の第 1 段（短距離のみ） |

### Mesh SDF vs Global SDF の使い分け

- **Mesh SDF**: メッシュ単位、高解像度（narrow-band） → 接触陰など細部に有効
- **Global SDF**: シーン全体のクリップマップ、低解像度 → 遠距離フォールバック

レイの最初は HZB → Mesh SDF → Global SDF と段階的にフォールバック。

---

## 7. 参考資料

- [S10] Wright / Heitz / Hillaire 2022 §4 — SW RT 設計
- Wright 2015, "Distance Field Soft Shadows" — Mesh Distance Field 原典
- Crassin et al. "Sparse Voxel GI" — Voxel 表現の比較先
- 関連: [[lumen_hw_rt]] / [[lumen_surface_cache]]

---

## 8. 相談用フック

- **理解度チェック**:
  - SDF とは何か → §2.1、表面までの最短距離関数
  - Sphere Tracing が安全な理由 → SDF 値ぶん進んでも表面に触れない数学的保証
  - Mesh SDF と Global SDF の役割分担 → §6
- **コード深掘り候補**:
  - `LumenScreenProbeTracing.cpp` の HZB → MeshSDF → GlobalSDF フォールバック分岐
  - `LumenMeshSDFCulling.usf` の AABB-Frustum テスト
  - `MeshSDFNotCoveredExpandSurfaceScale` の数学的根拠（Foliage two-sided 補正）
- **未読箇所**:
  - S10 §4.4 Heightfield Tracing
  - Global SDF の毎フレーム再ラスタライズパス（`DistanceFieldGlobalIllumination.cpp` 系）
- **次の派生**:
  - Heightfield (Landscape) トレース → [[lumen_heightfield_trace]]（未着手）
  - SW RT による Reflection Tracing → [[lumen_reflections_sw]]（未着手）
  - Mesh SDF オフライン生成パス → [[../Geometry/...]] 系
