# b: Velocity Buffer（RenderVelocities）

- 対象ファイル: `VelocityRendering.h/.cpp`
- 概要: [[13_depthprepass_overview]]

---

## 概要

**Velocity Buffer** は各ピクセルのスクリーン空間での移動量（Motion Vector）を格納する。  
- **TSR / TAA**: テンポラルアンチエイリアシングのリプロジェクションに使用  
- **Motion Blur**: 被写体ブラーの強度と方向を決定  
- **MegaLights / Lumen**: テンポラル蓄積のリプロジェクションにも利用  

---

## EVelocityPass（Velocity パス種別）

```cpp
// VelocityRendering.h:20
enum class EVelocityPass : uint32
{
    Opaque = 0,              // 不透明オブジェクトの Velocity
    Translucent,             // 半透明オブジェクトの Velocity（TranslucentBasePass 後）
    TranslucentClippedDepth, // OpacityMask でクリップされた半透明
    Count
};
```

---

## FVelocityRendering（ユーティリティ構造体）

```cpp
// VelocityRendering.h:37
struct FVelocityRendering
{
    // Velocity Buffer のピクセルフォーマット取得
    static EPixelFormat GetFormat(EShaderPlatform ShaderPlatform);
    // R16G16_SNORM または R16G16_FLOAT

    // テクスチャ作成フラグ取得
    static ETextureCreateFlags GetCreateFlags(EShaderPlatform ShaderPlatform);

    // 深度パスで Velocity を同時出力できるか判定
    // r.BasePassOutputsVelocity が有効なプラットフォームで true
    static bool DepthPassCanOutputVelocity(ERHIFeatureLevel::Type FeatureLevel);

    // BasePass で Velocity を同時出力できるか判定
    static bool BasePassCanOutputVelocity(EShaderPlatform ShaderPlatform);

    // 並列 Velocity パスを使用するか判定
    static bool IsParallelVelocity(EShaderPlatform ShaderPlatform);

    // プリミティブが Velocity 出力が必要なサイズかを判定
    static bool PrimitiveHasVelocityForView(const FViewInfo&, const FPrimitiveSceneProxy*);
};
```

---

## FVelocityMeshProcessor

```cpp
// VelocityRendering.h:63
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

    // スクリーン空間サイズ閾値判定
    static bool PrimitiveHasVelocityForView(
        const FViewInfo& View,
        const FPrimitiveSceneProxy* PrimitiveSceneProxy);

protected:
    EDepthDrawingMode EarlyZPassMode = DDM_None;
    EMeshPass::Type MeshPassType = EMeshPass::Type::Num;
};
```

---

## RenderVelocities() 内部フロー

```
AddVelocityPass() / RenderVelocities()          VelocityRendering.cpp
  │
  ├─ [条件チェック]
  │   ShouldRenderVelocities() = TSR / TAA / Motion Blur が有効
  │   FVelocityRendering::IsVelocityPassSupported(Platform)
  │
  ├─ [Velocity テクスチャ確保]
  │   FVelocityRendering::GetRenderTargetDesc()
  │   → R16G16_FLOAT テクスチャ（フォーマットはプラットフォーム依存）
  │
  ├─ [Opaque Velocity パス]
  │   FVelocityMeshProcessor(Opaque) で描画
  │     AddMeshBatch():
  │       PrimitiveHasVelocityForView() → 画面上サイズ閾値チェック
  │       動的プリミティブ（移動オブジェクト）:
  │         FVelocityVS: CurrentLocalToWorld × ViewProj → ScreenPos_curr
  │         FVelocityVS: PrevLocalToWorld × PrevViewProj → ScreenPos_prev
  │         FVelocityPS: (ScreenPos_curr - ScreenPos_prev) を R16G16 に書き込み
  │       静的プリミティブ（動かないもの）:
  │         スキップ（Velocity = 0 のまま）
  │
  ├─ [Skinned Mesh（スケルタルメッシュ）]
  │   FVelocityMeshProcessor で同様に処理
  │   ただしボーン行列（前フレーム分）も考慮
  │   → SkinnedMesh の頂点ブレンド後の位置差分を計算
  │
  ├─ [Nanite Velocity]
  │   Nanite::RenderVelocities()
  │   → Nanite VisBuffer から Compute Shader でMotion Vector を生成
  │   → InstanceTransform 変化のある Nanite プリミティブのみ出力
  │
  └─ [Translucent Velocity（後フェーズ）]
      FVelocityMeshProcessor(Translucent) で半透明の Velocity を出力
      → TranslucentBasePass の後に実行

【Velocity の解釈】
  Velocity テクスチャ値 = (0.5, 0.5) → 移動なし（エンコードオフセット）
  Velocity.x = (ScreenPos_curr.x - ScreenPos_prev.x) * 0.5 + 0.5
  Velocity.y = (ScreenPos_curr.y - ScreenPos_prev.y) * 0.5 + 0.5
  → TSR/TAA がリプロジェクション時に (Velocity * 2 - 1) でデコード
```

---

## 関連リファレンス

- [[ref_velocity_rendering]] — `FVelocityVS` / `FVelocityPS` / パラメータ詳細
- [[a_depth_rendering]] — Depth PrePass（同フレームに実行）
