# REF: FMeshDrawCommand

- 対象ファイル: `MeshDrawCommands.h/.cpp`
- 関連Details: [[b_mpp_drawcommand]]

---

## FMeshDrawCommand — 全メンバ

```cpp
class FMeshDrawCommand
{
public:
    // ─── シェーダーバインディング ───
    FMeshDrawShaderBindings ShaderBindings;

    // ─── 頂点ストリーム ───
    FVertexInputStreamArray VertexStreams;

    // ─── インデックスバッファ ───
    FRHIBuffer* IndexBuffer;  // nullptr = 非インデックス描画

    // ─── PSO ───
    FGraphicsMinimalPipelineStateId CachedPipelineId;

    // ─── 描画引数 ───
    uint32 FirstIndex;      // インデックスバッファの開始位置
    uint32 NumPrimitives;   // 0 = DrawIndirect
    uint32 NumInstances;

    union {
        struct { uint32 BaseVertexIndex; uint32 NumVertices; } VertexParams;
        struct { FRHIBuffer* Buffer; uint32 Offset; } IndirectArgs;
    };

    // ─── 追加データ ───
    int8  PrimitiveIdStreamIndex;  // GPUScene InstanceData ストリームインデックス
    uint8 StencilRef;
    EPrimitiveType PrimitiveType : PT_NumBits;

#if MESH_DRAW_COMMAND_DEBUG_DATA
    FMeshDrawCommandDebugData DebugData;  // デバッグ情報（非 Shipping のみ）
#endif

    // ─── 初期化 ───
    void InitializeShaderBindings(const FMeshProcessorShaders& Shaders);
    void SetStencilRef(uint32 InStencilRef);

    // ─── 確定 ───
    void SetDrawParametersAndFinalize(
        const FMeshBatch& MeshBatch, int32 BatchElementIndex,
        FGraphicsMinimalPipelineStateId PipelineId,
        const FMeshProcessorShaders* ShadersForDebugging);

    void Finalize(
        FGraphicsMinimalPipelineStateId PipelineId,
        const FMeshProcessorShaders* ShadersForDebugging);

    // ─── ダイナミックインスタンシング ───
    bool MatchesForDynamicInstancing(const FMeshDrawCommand& Rhs) const;
    uint32 GetDynamicInstancingHash() const;

    // ─── 発行 ───
    static bool SubmitDrawBegin(
        const FMeshDrawCommand& MeshDrawCommand,
        const FGraphicsMinimalPipelineStateSet& GraphicsMinimalPipelineStateSet,
        FRHIBuffer* SceneArgs,
        uint32 SceneInstanceOffset,
        uint32 NumInstances,
        FRHICommandList& RHICmdList);

    static void SubmitDrawEnd(
        const FMeshDrawCommand& MeshDrawCommand,
        uint32 NumInstances,
        FRHICommandList& RHICmdList);

    static bool SubmitDraw(
        const FMeshDrawCommand& MeshDrawCommand,
        const FGraphicsMinimalPipelineStateSet& PipelineStateSet,
        FRHIBuffer* SceneArgs,
        uint32 SceneInstanceOffset,
        uint32 NumInstances,
        FRHICommandList& RHICmdList);
};
```

---

## FMeshDrawCommandSortKey

```cpp
class FMeshDrawCommandSortKey
{
public:
    static FMeshDrawCommandSortKey Default;

    union {
        uint64 PackedData;

        // BasePass ソート（PSO 変更を最小化）
        struct {
            uint64 VertexShaderHash  : 16;  // VS のハッシュ上位 16 ビット
            uint64 PixelShaderHash   : 32;  // PS のハッシュ下位 32 ビット
            uint64 Masked            : 1;   // Masked マテリアルを後ろに
            uint64 PipelineId        : 15;
        } BasePass;

        // 半透明ソート（後ろ→前の Z ソート）
        struct {
            uint64 MeshIdInPrimitive : 16;  // プリミティブ内メッシュ番号
            uint64 Distance          : 32;  // カメラ距離（逆順: 遠→近）
            uint64 Priority          : 16;  // マテリアル ETranslucencySortPriority
        } Translucent;
    };

    // ファクトリ
    static FMeshDrawCommandSortKey GetDefaultTranslucent(
        const FPrimitiveSceneProxy* PrimitiveSceneProxy,
        const FMeshBatch& MeshBatch,
        const FMaterial& Material);

    bool operator<(const FMeshDrawCommandSortKey& Rhs) const
    {
        return PackedData < Rhs.PackedData;
    }
};
```

---

## FMeshDrawShaderBindings

```cpp
class FMeshDrawShaderBindings
{
public:
    // 使用シェーダーに基づき領域を確保
    void Initialize(const FMeshProcessorShaders& Shaders);

    // シェーダー周波数ごとのバインディングアクセサ
    FMeshDrawSingleShaderBindings GetSingleShaderBindings(
        EShaderFrequency Frequency, int32& DataOffset);

    // RHI コマンドリストへの適用
    void SetOnCommandList(FRHICommandList& RHICmdList,
        FRHIVertexDeclaration* VertexDeclaration = nullptr) const;

    void SetOnCommandListForCompute(FRHIComputeCommandList& RHICmdList,
        const TShaderRef<FShader>& Shader) const;

    // バインディング比較（動的インスタンシング判定）
    bool operator==(const FMeshDrawShaderBindings& Rhs) const;
    bool operator!=(const FMeshDrawShaderBindings& Rhs) const;

    uint32 GetDynamicInstancingHash() const;
};
```

---

## FMeshDrawSingleShaderBindings

```cpp
class FMeshDrawSingleShaderBindings : public FMeshDrawShaderBindingsLayout
{
public:
    // Uniform Buffer
    template<typename UniformBufferStructType>
    void Add(const TShaderUniformBufferParameter<UniformBufferStructType>& Parameter,
             const TUniformBufferRef<UniformBufferStructType>& Value);

    void Add(const FShaderUniformBufferParameter& Parameter,
             FRHIUniformBuffer* Value);

    // SRV
    void Add(FShaderResourceParameter Parameter, FRHIShaderResourceView* Value);

    // サンプラー
    void Add(FShaderResourceParameter SamplerParameter,
             FRHISamplerState* SamplerStateRHI);

    // テクスチャ
    void Add(FShaderResourceParameter TextureParameter, FRHITexture* TextureRHI);

    // スカラー / ベクター（ルート定数 / インライン定数）
    template<class ParameterType>
    void Add(FShaderParameter Parameter, const ParameterType& Value);
};
```

---

## FCachedMeshDrawCommands — 静的キャッシュ

```cpp
// 1 プリミティブあたりのパス別キャッシュエントリ
struct FCachedMeshDrawCommandInfo
{
    FMeshDrawCommandSortKey SortKey;
    int32 CommandIndex;              // FCachedPassMeshDrawList::MeshDrawCommands のインデックス
    int32 StateBucketId;
    EFVisibleMeshDrawCommandFlags CommandFlags;
};

// パス全体のキャッシュ（FCachedPassMeshDrawList が EMeshPass ごとに存在）
struct FCachedPassMeshDrawList
{
    TArray<FMeshDrawCommand> MeshDrawCommands;
    TBitArray<>              CommandProcessedFrameNumber;
};
```

---

## FVisibleMeshDrawCommand — フレームあたりの可視コマンド

```cpp
struct FVisibleMeshDrawCommand
{
    const FMeshDrawCommand* MeshDrawCommand;
    FMeshDrawCommandSortKey SortKey;
    int32 StateBucketId;
    uint32 RunArray[2];              // ダイナミックインスタンシング用ランレングス
    EFVisibleMeshDrawCommandFlags Flags;
    int8 PrimitiveIdStreamIndex;
};

// フレーム一時配列型
using FMeshCommandOneFrameArray = TArray<FVisibleMeshDrawCommand, SceneRenderingAllocator>;
```
