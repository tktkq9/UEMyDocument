# ref: BasePass レンダラー クラス群

- 対象: `BasePassRendering.h` / `BasePassRendering.inl`
- Details: [[a_basepass_pipeline]] | [[c_basepass_material]]

---

## TBasePassMeshProcessor

```cpp
// BasePassRendering.h
class FBasePassMeshProcessor : public FSceneRenderingAllocatorObject<FBasePassMeshProcessor>,
                                public FMeshPassProcessor
{
public:
    FBasePassMeshProcessor(
        EMeshPass::Type InMeshPassType,
        const FScene* InScene,
        ERHIFeatureLevel::Type InFeatureLevel,
        const FSceneView* InViewIfDynamicMeshCommand,
        const FMeshPassProcessorRenderState& InDrawRenderState,
        FMeshPassDrawListContext* InDrawListContext,
        EFlags InFlags,
        ETranslucencyPass::Type InTranslucencyPassType);

    // キーメソッド: メッシュバッチから DrawCommand を生成
    virtual void AddMeshBatch(
        const FMeshBatch& RESTRICT MeshBatch,
        uint64 BatchElementMask,
        const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,
        int32 StaticMeshId = -1) override;

private:
    ETranslucencyPass::Type TranslucencyPassType;
    EDepthDrawingMode EarlyZPassMode;
    bool bTranslucentBasePass;
    bool bEnableReceiveDecalOutput;  // DBuffer デカールを受け取るか
    bool bForcePassDrawRenderState;
};
```

---

## TBasePassVertexShaderPolicyParamType

```cpp
// BasePassRendering.h / .inl
template<typename LightMapPolicyType>
class TBasePassVertexShaderPolicyParamType : public FMeshMaterialShader
{
public:
    // IMPLEMENT_MATERIAL_SHADER_TYPE で実装される
    static bool ShouldCompilePermutation(const FMeshMaterialShaderPermutationParameters&);
    static void ModifyCompilationEnvironment(const FMaterialShaderPermutationParameters&,
                                             FShaderCompilerEnvironment&);

    void GetShaderBindings(const FScene*, ERHIFeatureLevel::Type, const FPrimitiveSceneProxy*,
        const FMaterialRenderProxy&, const FMaterial&, const FMeshPassProcessorRenderState&,
        const TBasePassShaderElementData<LightMapPolicyType>&,
        FMeshDrawSingleShaderBindings&) const;
};
```

---

## TBasePassPixelShaderPolicyParamType

```cpp
template<typename LightMapPolicyType>
class TBasePassPixelShaderPolicyParamType : public FMeshMaterialShader
{
public:
    static bool ShouldCompilePermutation(const FMeshMaterialShaderPermutationParameters&);
    static void ModifyCompilationEnvironment(const FMaterialShaderPermutationParameters&,
                                             FShaderCompilerEnvironment&);

    // シェーダーパラメータのバインド
    void GetShaderBindings(const FScene*, ERHIFeatureLevel::Type, const FPrimitiveSceneProxy*,
        const FMaterialRenderProxy&, const FMaterial&, const FMeshPassProcessorRenderState&,
        const TBasePassShaderElementData<LightMapPolicyType>&,
        FMeshDrawSingleShaderBindings&) const;

    // FOpaqueBasePassUniformParameters をバインド
    LAYOUT_FIELD(FShaderUniformBufferParameter, PassUniformBuffer);
};
```

---

## 主要関数一覧

| 関数 | ファイル | 役割 |
|-----|---------|------|
| `RenderBasePass()` | `BasePassRendering.cpp:1071` | 静的エントリポイント |
| `RenderBasePassInternal()` | `BasePassRendering.cpp:1449` | RDG Pass 発行・並列描画 |
| `FBasePassMeshProcessor::AddMeshBatch()` | `BasePassRendering.cpp` | DrawCommand 生成 |
| `GetUniformLightMapPolicy()` | `BasePassRendering.cpp` | ライトマップポリシー選択 |
| `SetupBasePassState()` | `BasePassRendering.cpp` | RenderState 設定 |

---

> [!note]- TBasePassMeshProcessor は static メソッド RenderBasePass の設計
> `RenderBasePass()` が `static` なのは、Custom Render Pass（SceneCapture 等）でも同じ実装を再利用するため。
> `this（FDeferredShadingSceneRenderer）` に依存せず `FDeferredShadingSceneRenderer& Renderer` を引数で受け取る。

> [!note]- シェーダー排列数の爆発
> `TBasePassPS<LightMapPolicyType, NumDynamicPointLights>` は LightMapPolicy × DynamicPointLights の組み合わせで多数生成される。
> `r.PSOPrecache.LightMapPolicyMode=1`（デフォルト）で `LMP_NO_LIGHTMAP` のみプリコンパイルしてビルド時間を削減。

> [!note]- Nanite との分担
> Nanite メッシュは `TBasePassMeshProcessor` を通らず、`NaniteShading::EmitGBuffer()` が直接 GBuffer に書き込む。
> `NaniteBasePassShadingCommands` として事前に生成された ShadingCommand リストを CS でディスパッチする。
