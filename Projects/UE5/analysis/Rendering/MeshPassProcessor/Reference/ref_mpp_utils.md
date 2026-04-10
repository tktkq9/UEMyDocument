# REF: MeshPassProcessor ユーティリティ

- 対象ファイル: `SimpleMeshDrawCommandPass.h`, `MeshPassUtils.h`, `MeshDrawCommandStats.h`
- 関連Details: [[d_mpp_passes]]

---

## AddSimpleMeshPass — 高レベルパス追加ヘルパー

```cpp
// 動的メッシュコマンドをまとめて RDG に追加する汎用ヘルパー
// MeshBatchesCallback でコマンドを登録し、PrologueCallback でビューポート等を設定
template <
    typename PassParametersType,
    typename AddMeshBatchesCallbackLambdaType,
    typename PassPrologueLambdaType>
void AddSimpleMeshPass(
    FRDGBuilder& GraphBuilder,
    PassParametersType* PassParametersIn,
    const FScene* Scene,
    const FSceneView& View,
    FInstanceCullingManager* InstanceCullingManager,
    FRDGEventName&& PassName,
    const ERDGPassFlags& PassFlags,
    AddMeshBatchesCallbackLambdaType AddMeshBatchesCallback,
    PassPrologueLambdaType PassPrologueCallback,
    bool bAllowIndirectArgsOverride = true);
```

### コールバック引数の型

```cpp
// AddMeshBatchesCallback の引数
void AddMeshBatchesCallback(FDynamicPassMeshDrawListContext* DrawListContext);

// PassPrologueCallback の引数
void PassPrologueCallback(FRHICommandList& RHICmdList);
```

### 使用例

```cpp
auto* PassParams = GraphBuilder.AllocParameters<FDepthPassParameters>();
PassParams->RenderTargets.DepthStencil = FDepthStencilBinding(
    SceneDepth, ERenderTargetLoadAction::EClear);

AddSimpleMeshPass(
    GraphBuilder, PassParams, Scene, View, InstanceCullingManager,
    RDG_EVENT_NAME("DepthPrepass"),
    ERDGPassFlags::Raster,
    [&](FDynamicPassMeshDrawListContext* DrawListContext)
    {
        FDepthPassMeshProcessor Processor(Scene, &View, DrawRenderState, DrawListContext);
        for (const FMeshBatch& Batch : View.DynamicMeshElements)
        {
            Processor.AddMeshBatch(Batch, /* BatchElementMask */ ~0ull, nullptr);
        }
    },
    [ViewRect](FRHICommandList& RHICmdList)
    {
        RHICmdList.SetViewport(
            ViewRect.Min.X, ViewRect.Min.Y, 0.0f,
            ViewRect.Max.X, ViewRect.Max.Y, 1.0f);
    });
```

---

## MeshPassUtils.h — レンダーステートユーティリティ

```cpp
// デフォルトレンダーステートを設定するユーティリティ

// DepthPass 用ステート（書き込みあり・アルファテストなし）
void SetupDepthPassState(FMeshPassProcessorRenderState& DrawRenderState);

// BasePass 用ステート
void SetupBasePassState(
    FExclusiveDepthStencil::Type BasePassDepthStencilAccess,
    const bool bShaderComplexity,
    FMeshPassProcessorRenderState& DrawRenderState);

// Translucency 用ステート（ブレンドモードに応じて切り替え）
void SetupTranslucencyState(
    ETranslucencyPass::Type TranslucencyPass,
    bool bIsMeshDecal,
    FMeshPassProcessorRenderState& DrawRenderState);

// Shadow Depth 用ステート
void SetupShadowDepthPassState(FMeshPassProcessorRenderState& DrawRenderState);
```

---

## EMeshPass — 関連 API

```cpp
// パス名を文字列に変換
inline const TCHAR* GetMeshPassName(EMeshPass::Type MeshPass);

// パスのビット数（上限チェック用）
// EMeshPass::Num <= (1 << EMeshPass::NumBits) であることを保証
// NumBits = 6 → 最大 64 パスまで
static_assert(EMeshPass::Num <= (1 << EMeshPass::NumBits));
```

---

## FMeshBatch — メッシュ描画要求

```cpp
// AddMeshBatch() に渡す構造体
struct FMeshBatch
{
    TArray<FMeshBatchElement, TInlineAllocator<1>> Elements;  // バッチ要素（複数 LOD 等）

    const FVertexFactory*          VertexFactory;
    const FMaterialRenderProxy*    MaterialRenderProxy;

    uint32 bDitheredLODTransition : 1;
    uint32 bReverseCulling        : 1;
    uint32 bDisableBackfaceCulling : 1;
    uint32 bWireframe             : 1;
    uint32 bUseForMaterial        : 1;    // マテリアルシェーダーを使うか
    uint32 bUseForDepthPass       : 1;
    uint32 bUseAsOccluder         : 1;

    uint8 LODIndex;
    uint8 SegmentIndex;
    EPrimitiveType Type;                  // PT_TriangleList 等
    uint32 MeshIdInPrimitive;             // プリミティブ内のメッシュ番号
    float  MeshScreenSizeSquared;         // スクリーン占有率（LOD 判定用）

    bool IsTranslucent(ERHIFeatureLevel::Type FeatureLevel) const;
};

// バッチ要素（実際の描画引数）
struct FMeshBatchElement
{
    const FRHIBuffer*         IndexBuffer;
    uint32                    FirstIndex;
    uint32                    NumPrimitives;
    uint32                    NumInstances;
    uint32                    MinVertexIndex;
    uint32                    MaxVertexIndex;
    int32                     BaseVertexIndex;
    const FRHIBuffer*         IndirectArgsBuffer;
    uint32                    IndirectArgsOffset;
    const FRHIBuffer*         InstanceRuns;    // ランベースインスタンス化
    uint32                    NumRuns;
    FUniformBufferRHIRef      PrimitiveUniformBuffer;
    const FPrimitiveSceneInfo* PrimitiveScene;
    float                     LODIndex;
    float                     MinScreenSize;
    float                     MaxScreenSize;
};
```

---

## FInstanceCullingManager — インスタンスカリング管理

```cpp
class FInstanceCullingManager
{
public:
    // GPU インスタンスカリングが有効か
    bool IsEnabled() const;

    // 追加（AddSimpleMeshPass 等に渡す）
    FInstanceCullingDrawParams* AddInstanceCullingDraw(
        const FMeshDrawCommand& DrawCommand,
        FRDGBuffer* DrawIndirectArgsBuffer, uint32 DrawIndirectArgsOffset,
        FRDGBuffer* InstanceDataBuffer, uint32 InstanceDataOffset,
        uint32 NumInstances);
};
```

---

## FMeshDrawCommandStats — デバッグ統計

`MESH_DRAW_COMMAND_DEBUG_DATA` 定義時のみ有効:

```cpp
struct FVisibleMeshDrawCommandStatsData
{
    int32 DrawCallCount;
    int32 MeshCount;
    int32 TotalInstanceCount;
    int32 DynamicInstanceCount;
};

// パスの統計を表示（r.MeshDrawCommands.LogCommandCounts 等）
void LogVisibleMeshDrawCommandStats(
    const FMeshCommandOneFrameArray& VisibleMeshDrawCommands,
    EMeshPass::Type MeshPassType,
    int32 ViewIndex);
```
