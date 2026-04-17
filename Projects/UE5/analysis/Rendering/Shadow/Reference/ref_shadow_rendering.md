# ref: FShadowDepthVS / FShadowDepthPS / FShadowDepthPassMeshProcessor

- 対象ファイル: `ShadowDepthRendering.h/.cpp` / `ShadowRendering.h`
- 概要: [[17_shadow_overview]]

---

## FShadowDepthPassMeshProcessor（ShadowRendering.h:115）

Shadow Depth Map を描画するための `FMeshPassProcessor` 派生クラス。  
Opaque / Masked / Nanite のいずれも同クラスで処理される。

```cpp
class FShadowDepthPassMeshProcessor
    : public FSceneRenderingAllocatorObject<FShadowDepthPassMeshProcessor>
    , public FMeshPassProcessor
{
public:
    FShadowDepthPassMeshProcessor(
        const FScene* Scene,
        const ERHIFeatureLevel::Type InFeatureLevel,
        const FSceneView* InViewIfDynamicMeshCommand,
        EShadowDepthType ShadowDepthType,          // Perspective / Orthographic / Cube
        FMeshPassDrawListContext* InDrawListContext);

    virtual void AddMeshBatch(...) override final;
    virtual void CollectPSOInitializers(...) override final;

    FMeshPassProcessorRenderState PassDrawRenderState;

private:
    bool TryAddMeshBatch(...);
    bool Process(
        const FMeshBatch& RESTRICT MeshBatch,
        uint64 BatchElementMask,
        int32 StaticMeshId,
        const FPrimitiveSceneProxy* PrimitiveSceneProxy,
        const FMaterialRenderProxy& MaterialRenderProxy,
        const FMaterial& MaterialResource,
        ERasterizerFillMode MeshFillMode,
        ERasterizerCullMode MeshCullMode);

    EShadowDepthType ShadowDepthType;
    EShadowMeshSelection MeshSelectionMask = EShadowMeshSelection::All;
};
```

---

## EShadowDepthType

```cpp
// ShadowDepthRendering.h
enum class EShadowDepthType : uint8
{
    None,
    Perspective,    // Spot Light: パース投影
    Orthographic,   // CSM (Directional Light): 平行投影
    Cube,           // Point Light: キューブマップ
};
```

---

## Shadow Depth シェーダー（ShadowDepthRendering.h）

### FShadowDepthVS（頂点シェーダー）

```cpp
// ShadowDepthRendering.h
class FShadowDepthVS : public FMeshMaterialShader
{
    DECLARE_SHADER_TYPE(FShadowDepthVS, MeshMaterial);

    // パーミューテーション
    class FPerspectiveCorrect : SHADER_PERMUTATION_BOOL("PERSPECTIVE_CORRECT_DEPTH");
    class FOutputDepth       : SHADER_PERMUTATION_BOOL("OUTPUT_DEPTH_TO_COLOR");
    class FPositionOnly      : SHADER_PERMUTATION_BOOL("POSITION_ONLY");
    using FPermutationDomain = TShaderPermutationDomain<
        FPerspectiveCorrect, FOutputDepth, FPositionOnly>;

    static bool ShouldCompilePermutation(
        const FMeshMaterialShaderPermutationParameters& Parameters);

    // ─── 処理内容 ─────────────────────────────────────────────────
    // TranslatedWorldToClipInnerMatrix で頂点変換
    // CSM:   Orthographic 投影（光源方向に平行）
    // Spot:  Perspective 投影（光源位置から錐台）
    // Point: Cube 面ごとに Perspective 投影
    //
    // FPositionOnly=true のとき: Position-Only ストリームを使用（軽量）
    //   → Opaque Mesh に適用
    // FPositionOnly=false のとき: フル頂点ストリーム
    //   → Masked Mesh（UV / Normal が必要）に適用
};
```

### FShadowDepthPS（ピクセルシェーダー）

```cpp
class FShadowDepthPS : public FMeshMaterialShader
{
    DECLARE_SHADER_TYPE(FShadowDepthPS, MeshMaterial);

    static bool ShouldCompilePermutation(
        const FMeshMaterialShaderPermutationParameters& Parameters)
    {
        // Masked マテリアルまたは Pixel Depth Offset がある場合のみコンパイル
        return !Parameters.MaterialParameters.bWritesEveryPixel
            || Parameters.MaterialParameters.bHasPixelDepthOffsetConnected;
    }

    // ─── 処理内容 ─────────────────────────────────────────────────
    // Masked マテリアル: OpacityMask < ClipValue → discard
    // Pixel Depth Offset: PDO 値で出力深度を調整
    //
    // Opaque で PDO なし: PSなし → 深度バッファに直接書き込み
};
```

---

## Shadow Atlas テクスチャフォーマット

| ライスト種別 | フォーマット | 説明 |
|----------|----------|------|
| Directional（CSM） | `D24S8` or `R32_FLOAT` | 複数カスケードをアトラスに格納 |
| Spot Light | `D24S8` | 個別テクスチャまたはアトラス |
| Point Light（キューブ）| `D24S8` | 6 面または CubeTexture |
| Per-Object Dynamic | `R32_FLOAT` | 動的シャドウ（小解像度）|
| Cached Static | `D24S8` | キャッシュ済み静的シャドウ |

---

## EShadowMeshSelection

```cpp
// ShadowRendering.h
enum class EShadowMeshSelection : uint8
{
    SM  = 1U << 0U,      // 従来 Shadow Map
    VSM = 1U << 1U,      // Virtual Shadow Map（GPU Scene Instance Culling 対応時）
    All = SM | VSM,
};
ENUM_CLASS_FLAGS(EShadowMeshSelection)
```

---

## RenderShadowDepthMaps() ディスパッチロジック

```
RenderShadowDepthMaps()                         ShadowDepthRendering.cpp
  │
  for each FProjectedShadowInfo:
  │
  ├─ [1] Shadow Atlas RT バインド
  │       ScissorRect = (X, Y, X+ResX, Y+ResY)
  │
  ├─ [2] Nanite メッシュ（bNaniteGeometry=true）
  │       Nanite::RenderShadowDepths(Shadow, View, ...)
  │         → NaniteShadowDepth.usf (Compute Shader)
  │         → VisBuffer の ClusterID/TriangleID を Shadow 空間に変換
  │         → ラスタライザを通さないため高速
  │
  ├─ [3] 非 Nanite Opaque
  │       FProjectedShadowInfo::RenderDepth()
  │         FShadowDepthPassMeshProcessor::AddMeshBatch()
  │           FShadowDepthVS (FPositionOnly=true)  // 位置専用ストリーム
  │           PS なし（深度のみ書き込み）
  │           → RHIDrawIndexedPrimitive()
  │
  ├─ [4] 非 Nanite Masked
  │       同上 + FShadowDepthPS でアルファテスト
  │         FShadowDepthVS (FPositionOnly=false)  // 全頂点ストリーム
  │         FShadowDepthPS (OpacityMask clip)
  │
  └─ [5] Cached Shadow（静的シャドウキャッシュ）
          CacheMode == SDCM_StaticPrimitivesOnly
          → 前フレームの Shadow テクスチャをコピーして再利用
          → 動的オブジェクトのみ追加描画

【Atlas UV 補正】
  テクセル座標 = (X + BorderSize, Y + BorderSize) からコンテンツ開始
  シェーダー内 Shadow UV:
    UV.xy = ShadowPos.xy * 0.5 + 0.5
    UV.xy = UV.xy * vec2(ResX, ResY) / AtlasSize
          + vec2(X + BorderSize, Y + BorderSize) / AtlasSize
```

---

## 関連リファレンス

- [[ref_projected_shadow_info]] — `FProjectedShadowInfo` 全メンバ・カスケード設定
- [[ref_shadow_projection]] — `TShadowProjectionPS` / PCF パターン
- [[b_shadow_depth]] — Shadow Depth 描画フロー詳細
