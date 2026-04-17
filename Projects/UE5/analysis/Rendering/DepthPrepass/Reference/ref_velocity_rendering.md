# ref: FVelocityMeshProcessor / FVelocityRendering / EVelocityPass

- 対象ファイル: `VelocityRendering.h/.cpp`
- 概要: [[13_depthprepass_overview]]

---

## EVelocityPass（VelocityRendering.h:19）

```cpp
enum class EVelocityPass : uint32
{
    Opaque = 0,               // 不透明メッシュの別途 Velocity パス
    Translucent,              // 半透明メッシュの Velocity / Depth パス（透明パス後）
    TranslucentClippedDepth,  // OpacityMask でクリップされたピクセル向け二次パス
    Count
};

EMeshPass::Type GetMeshPassFromVelocityPass(EVelocityPass VelocityPass);
// → EMeshPass::Velocity / TranslucentVelocity / TranslucentVelocityClippedDepth
```

---

## FVelocityRendering（VelocityRendering.h:36）

ユーティリティ構造体（全 static メソッド）。  
Velocity バッファのフォーマット・ケイパビリティ判定を提供。

```cpp
struct FVelocityRendering
{
    // Velocity バッファのピクセルフォーマットを返す
    // → R16G16_FLOAT（通常）/ R16G16B16A16_FLOAT（VR / Multi-view）
    static EPixelFormat GetFormat(EShaderPlatform ShaderPlatform);

    // Velocity バッファのテクスチャ作成フラグ（UAV 対応等）
    static ETextureCreateFlags GetCreateFlags(EShaderPlatform ShaderPlatform);

    // Velocity バッファの RDGTextureDesc を返す
    static FRDGTextureDesc GetRenderTargetDesc(
        EShaderPlatform ShaderPlatform,
        FIntPoint Extent,
        const bool bRequireMultiView = false);

    // Velocity パスがサポートされているかどうか
    static bool IsVelocityPassSupported(EShaderPlatform ShaderPlatform);

    // Depth Pass 中に Velocity を出力できるかどうか
    // → SM6 以上 + BasePass で Velocity 不可の場合に有効
    static bool DepthPassCanOutputVelocity(ERHIFeatureLevel::Type FeatureLevel);

    // BasePass 中に Velocity を出力できるかどうか
    // → GBuffer レイアウトに Velocity チャンネルが含まれる場合
    static bool BasePassCanOutputVelocity(EShaderPlatform ShaderPlatform);

    // Parallel Velocity 描画が使えるかどうか
    static bool IsParallelVelocity(EShaderPlatform ShaderPlatform);
};
```

---

## FVelocityMeshProcessor（VelocityRendering.h:63）

Velocity パスの基底 MeshPassProcessor。  
`FOpaqueVelocityMeshProcessor` と `FTranslucentVelocityMeshProcessor` が派生する。

```cpp
class FVelocityMeshProcessor : public FMeshPassProcessor
{
public:
    FVelocityMeshProcessor(
        EMeshPass::Type MeshPassType,
        const FScene* Scene,
        ERHIFeatureLevel::Type FeatureLevel,
        const FSceneView* InViewIfDynamicMeshCommand,
        const FMeshPassProcessorRenderState& InPassDrawRenderState,
        FMeshPassDrawListContext* InDrawListContext);

    FMeshPassProcessorRenderState PassDrawRenderState;

    // ビューに対してプリミティブが Velocity を出力するか判定
    // → スクリーンスペースサイズが閾値以下なら出力しない
    static bool PrimitiveHasVelocityForView(
        const FViewInfo& View,
        const FPrimitiveSceneProxy* PrimitiveSceneProxy);

protected:
    bool Process(
        const FMeshBatch& MeshBatch,
        uint64 BatchElementMask,
        int32 StaticMeshId,
        const FPrimitiveSceneProxy* PrimitiveSceneProxy,
        const FMaterialRenderProxy& MaterialRenderProxy,
        const FMaterial& MaterialResource,
        ERasterizerFillMode MeshFillMode,
        ERasterizerCullMode MeshCullMode);

    EDepthDrawingMode EarlyZPassMode = DDM_None;
    EMeshPass::Type MeshPassType = EMeshPass::Type::Num;
};
```

---

## FOpaqueVelocityMeshProcessor（VelocityRendering.h:106）

```cpp
class FOpaqueVelocityMeshProcessor
    : public FSceneRenderingAllocatorObject<FOpaqueVelocityMeshProcessor>
    , public FVelocityMeshProcessor
{
public:
    FOpaqueVelocityMeshProcessor(
        const FScene* Scene,
        ERHIFeatureLevel::Type FeatureLevel,
        const FSceneView* InViewIfDynamicMeshCommand,
        const FMeshPassProcessorRenderState& InPassDrawRenderState,
        FMeshPassDrawListContext* InDrawListContext,
        EDepthDrawingMode InEarlyZPassMode);

    // プリミティブが任意のフレームで Velocity を持てるか判定
    static bool PrimitiveCanHaveVelocity(
        EShaderPlatform ShaderPlatform,
        const FPrimitiveSceneProxy* PrimitiveSceneProxy);
    static bool PrimitiveCanHaveVelocity(
        EShaderPlatform ShaderPlatform,
        bool bDrawVelocity,
        bool bHasStaticLighting);
};
```

---

## Velocity シェーダー

```cpp
// VelocityShader.usf の主要シェーダー

// 頂点シェーダー
// → 現フレームと前フレームの ClipPos を計算
//   CurrentClipPos = TranslatedWorldToClip × CurrentWorldPos
//   PreviousClipPos = PrevTranslatedWorldToClip × PreviousWorldPos
//   out:  CurrentClipPos, PreviousClipPos → PS へ

// ピクセルシェーダー
// → VelocityBuffer への書き込み
//   Velocity.xy = (CurrentNDC.xy - PreviousNDC.xy) × 0.5
//               = MotionVector in NDC space
//   出力フォーマット: R16G16_FLOAT
//                     0.5 + delta（中央が0速度）
//
// Skinned Mesh: 前フレームの Bone Matrix を使って PreviousWorldPos を計算
// Static Mesh:  Actor の前フレーム Transform（FPreviousLocalToWorld）を使用
```

---

## Velocity エンコーディング

```
Velocity の格納形式（R16G16_FLOAT）:
  R = 0.5 + MotionVector.x  （-0.5〜0.5 → 0.0〜1.0）
  G = 0.5 + MotionVector.y

デコード（TSR / TAA / Motion Blur シェーダー内）:
  MotionVector.xy = VelocityTexture.Sample(UV).xy - 0.5

  MotionVector = 0 （静止）のとき → R=0.5, G=0.5 を書き込む
```

---

## Velocity バッファの利用先

| システム | 用途 |
|---------|------|
| TSR（Temporal Super Resolution） | スクリーン空間再投影・アンチエイリアシング |
| TAA（Temporal Anti-Aliasing） | ジッタキャンセル・時間的平均化 |
| Motion Blur | 後処理 Motion Blur の Tile 計算 |
| Lumen Reflection | 反射の時間的再利用（Screen Space Reprojection）|

---

## 関連リファレンス

- [[ref_depth_rendering]] — `TDepthOnlyVS` / `FDepthPassMeshProcessor`
- [[b_velocity_rendering]] — RenderVelocities() 詳細フロー
