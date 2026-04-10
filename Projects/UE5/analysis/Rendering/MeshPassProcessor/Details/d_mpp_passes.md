# D: EMeshPass enum と各パスの詳細

- 対象: `MeshPassProcessor.h`, `SimpleMeshDrawCommandPass.h`
- 上位: [[11_mpp_overview]]
- Reference: [[ref_mpp_utils]]

---

## EMeshPass — 全パス一覧

```cpp
namespace EMeshPass {
    enum Type : uint8 {
        // ─── 深度・プリパス ───
        DepthPass,                          // Z プリパス（不透明）
        SecondStageDepthPass,               // 2 段階深度パス（Nanite 後処理）

        // ─── GBuffer ───
        BasePass,                           // GBuffer 書き込み（非 Nanite）
        AnisotropyPass,                     // 異方性マテリアル専用パス
        SkyPass,                            // スカイボックス

        // ─── 水面 ───
        SingleLayerWaterPass,               // Single Layer Water（水面シェーディング）
        SingleLayerWaterDepthPrepass,       // 水面深度プリパス

        // ─── シャドウ ───
        CSMShadowDepth,                     // カスケードシャドウマップ
        VSMShadowDepth,                     // Virtual Shadow Maps
        OnePassPointLightShadowDepth,       // ポイントライトシャドウ（1パス Cubemap）

        // ─── 歪み・ベロシティ ───
        Distortion,                         // 歪みパス（屈折マテリアル）
        Velocity,                           // 不透明ベロシティ
        TranslucentVelocity,                // 半透明ベロシティ
        TranslucentVelocityClippedDepth,    // 深度クリップ付き半透明ベロシティ

        // ─── 半透明 ───
        TranslucencyStandard,               // 標準半透明（手前から奥）
        TranslucencyStandardModulate,       // Modulate ブレンド
        TranslucencyAfterDOF,               // DOF 後半透明（独立ぼけ）
        TranslucencyAfterDOFModulate,
        TranslucencyAfterMotionBlur,        // モーションブラー後
        TranslucencyHoldout,                // Holdout（背景透過）
        TranslucencyAll,                    // 全半透明（外部レンダリング用）

        // ─── デバッグ / 可視化 ───
        LightmapDensity,                    // ライトマップ密度表示
        DebugViewMode,                      // デバッグビューモード（複数モード）
        CustomDepth,                        // カスタム深度（アウトライン等）

        // ─── モバイル ───
        MobileBasePassCSM,                  // モバイル BasePass（CSM シャドウあり）

        // ─── Virtual Texture ───
        VirtualTexture,                     // Virtual Texture 書き込み

        // ─── Lumen ───
        LumenCardCapture,                   // Lumen Surface Cache カード更新
        LumenCardNanite,                    // Lumen 用 Nanite シェーディング
        LumenTranslucencyRadianceCacheMark, // Lumen 半透明 Radiance Cache マーク
        LumenFrontLayerTranslucencyGBuffer, // Lumen 半透明最前面レイヤー GBuffer

        // ─── LOD フェード ───
        DitheredLODFadingOutMaskPass,       // ディザ LOD フェードマスク（RT シャドウ用）

        // ─── Nanite ───
        NaniteMeshPass,                     // Nanite を通過するメッシュ

        // ─── メッシュデカール ───
        MeshDecal_DBuffer,                  // DBuffer デカール
        MeshDecal_SceneColorAndGBuffer,
        MeshDecal_SceneColorAndGBufferNoNormal,
        MeshDecal_SceneColor,
        MeshDecal_AmbientOcclusion,

        // ─── 水面情報 ───
        WaterInfoTextureDepthPass,
        WaterInfoTexturePass,

#if WITH_EDITOR
        // ─── エディタ専用 ───
        HitProxy,           // マウスクリック判定
        HitProxyOpaqueOnly,
        EditorLevelInstance, // レベルインスタンス境界
        EditorSelection,    // 選択アウトライン
#endif

        Num,
        NumBits = 6,   // Num を 6 bit に収める
    };
}
```

---

## 各パスのキャッシュ戦略

| パス | 静的キャッシュ | 説明 |
|------|-------------|------|
| `DepthPass` | あり | 不透明メッシュは毎フレーム再利用 |
| `BasePass` | あり | GBuffer 書き込み |
| `CSMShadowDepth` | あり | ライト位置が変わらない限り再利用 |
| `VSMShadowDepth` | あり | VSM 専用キャッシュ（CacheManager が管理）|
| `Velocity` | あり | 動くオブジェクトのみ動的 |
| `TranslucencyStandard` | なし | 半透明は毎フレーム動的生成 |
| `LumenCardCapture` | なし | Lumen カードは変化分のみ動的 |
| `CustomDepth` | あり | |
| `HitProxy` (Editor) | なし | エディタのみ、毎フレーム |

---

## AddSimpleMeshPass — 簡易パス追加ヘルパー

動的メッシュコマンドを追加するための高レベルAPI:

```cpp
template <typename PassParametersType,
          typename AddMeshBatchesCallbackLambdaType,
          typename PassPrologueLambdaType>
void AddSimpleMeshPass(
    FRDGBuilder& GraphBuilder,
    PassParametersType* PassParameters,
    const FScene* Scene,
    const FSceneView& View,
    FInstanceCullingManager* InstanceCullingManager,
    FRDGEventName&& PassName,
    const ERDGPassFlags& PassFlags,
    AddMeshBatchesCallbackLambdaType AddMeshBatchesCallback,
    PassPrologueLambdaType PassPrologueCallback,
    bool bAllowIndirectArgsOverride = true);
```

### 使用例

```cpp
FMyPassParameters* PassParams = GraphBuilder.AllocParameters<FMyPassParameters>();
PassParams->RenderTargets[0] = FRenderTargetBinding(OutputTex, ERenderTargetLoadAction::EClear);

AddSimpleMeshPass(
    GraphBuilder, PassParams, Scene, View, InstanceCullingManager,
    RDG_EVENT_NAME("MyDepthPass"),
    ERDGPassFlags::Raster,
    // メッシュ追加ラムダ
    [&](FDynamicPassMeshDrawListContext* DrawListContext)
    {
        FMyMeshPassProcessor Processor(Scene, &View, DrawRenderState, DrawListContext);
        for (const FMeshBatch& Mesh : DynamicMeshes)
        {
            Processor.AddMeshBatch(Mesh, BatchElementMask, nullptr);
        }
    },
    // プロローグラムダ（ビューポート設定等）
    [ViewRect](FRHICommandList& RHICmdList)
    {
        RHICmdList.SetViewport(ViewRect.Min.X, ViewRect.Min.Y, 0.0f,
                               ViewRect.Max.X, ViewRect.Max.Y, 1.0f);
    });
```

---

## DepthPass の典型的な実装パターン

```cpp
class FDepthPassProcessor : public FMeshPassProcessor
{
public:
    void AddMeshBatch(
        const FMeshBatch& MeshBatch, uint64 BatchElementMask,
        const FPrimitiveSceneProxy* Proxy, int32 StaticMeshId) override
    {
        const FMaterialRenderProxy* MatProxy = MeshBatch.MaterialRenderProxy;
        const FMaterial& Mat = MatProxy->GetMaterialWithFallback(FeatureLevel, MatProxy);

        // Masked マテリアルはピクセルシェーダーが必要
        const bool bMasked = Mat.GetBlendMode() == BLEND_Masked;

        TMeshProcessorShaders<FPositionOnlyDepthVS, FDepthOnlyPS> Shaders;
        Shaders.VertexShader = Mat.GetShader<FPositionOnlyDepthVS>(
            MeshBatch.VertexFactory->GetType(), 0, /* bPositionOnly */ !bMasked);
        if (bMasked)
            Shaders.PixelShader = Mat.GetShader<FDepthOnlyPS>(0);

        FMeshPassProcessorRenderState PassState;
        PassState.SetDepthStencilState(
            TStaticDepthStencilState<true, CF_DepthNearOrEqual>::GetRHI());
        PassState.SetBlendState(TStaticBlendState<>::GetRHI());

        BuildMeshDrawCommands(
            MeshBatch, BatchElementMask, Proxy, *MatProxy, Mat, PassState, Shaders,
            FM_Solid, CM_CCW,
            FMeshDrawCommandSortKey::Default,
            bMasked ? EMeshPassFeatures::Default : EMeshPassFeatures::PositionOnly,
            FMeshMaterialShaderElementData());
    }
};
```

---

## GetMeshPassName — パス名文字列変換

```cpp
inline const TCHAR* GetMeshPassName(EMeshPass::Type MeshPass)
{
    switch (MeshPass)
    {
    case EMeshPass::DepthPass:           return TEXT("DepthPass");
    case EMeshPass::BasePass:            return TEXT("BasePass");
    case EMeshPass::CSMShadowDepth:      return TEXT("CSMShadowDepth");
    case EMeshPass::VSMShadowDepth:      return TEXT("VSMShadowDepth");
    // ...
    }
}
```
