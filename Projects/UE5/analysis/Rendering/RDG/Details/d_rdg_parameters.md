# D: RDG シェーダーパラメータ・Blackboard・マクロ

- 対象: `RenderGraphParameter.h`, `RenderGraphBlackboard.h`, `RenderGraphEvent.h`, `ShaderParameterMacros.h`
- 上位: [[10_rdg_overview]]
- Reference: [[ref_rdg_utils]]

---

## シェーダーパラメータ構造体の定義マクロ

パス間でリソースを受け渡すための構造体を宣言するマクロ群。  
RDG はこの構造体を解析してリソース依存グラフを自動構築する。

```cpp
// 構造体の宣言（RENDER_GRAPH_PARAMETER_STRUCT: RDG 用の特殊マクロ）
BEGIN_SHADER_PARAMETER_STRUCT(FMyPassParameters, )
    // ─── テクスチャ ───
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D,      InTexture)
    SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture2D,  InTextureSRV)
    SHADER_PARAMETER_RDG_TEXTURE_UAV(Texture2D,  OutTextureUAV)
    SHADER_PARAMETER_RDG_TEXTURE_ARRAY(Texture2D, InTextureArray, [4])

    // ─── バッファ ───
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<FMyData>, InBuffer)
    SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<FMyData>, OutBuffer)

    // ─── Uniform Buffer ───
    SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneTextureUniformParameters, SceneTextures)

    // ─── スカラー / ベクター ───
    SHADER_PARAMETER(float,    MyFloat)
    SHADER_PARAMETER(FVector2f, UVScale)
    SHADER_PARAMETER(uint32,   NumElements)

    // ─── レンダーターゲット（Raster パスのみ）───
    RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()
```

### マクロ一覧

| マクロ | 対応シェーダー型 | HLSL 型 |
|--------|----------------|---------|
| `SHADER_PARAMETER_RDG_TEXTURE` | `FRDGTextureRef` | `Texture2D` 等 |
| `SHADER_PARAMETER_RDG_TEXTURE_SRV` | `FRDGTextureSRVRef` | `Texture2D` |
| `SHADER_PARAMETER_RDG_TEXTURE_UAV` | `FRDGTextureUAVRef` | `RWTexture2D` |
| `SHADER_PARAMETER_RDG_BUFFER_SRV` | `FRDGBufferSRVRef` | `StructuredBuffer<>` |
| `SHADER_PARAMETER_RDG_BUFFER_UAV` | `FRDGBufferUAVRef` | `RWStructuredBuffer<>` |
| `SHADER_PARAMETER_RDG_UNIFORM_BUFFER` | `TRDGUniformBufferRef<T>` | `cbuffer` |
| `SHADER_PARAMETER_STRUCT_REF` | `TUniformBufferRef<T>` | `cbuffer` |
| `SHADER_PARAMETER` | プリミティブ型 | `float`, `uint` 等 |
| `RENDER_TARGET_BINDING_SLOTS` | `FRenderTargetBindingSlots` | — |

---

## レンダーターゲットの設定

```cpp
// Raster パスでは RENDER_TARGET_BINDING_SLOTS() が必要
auto* PassParams = GraphBuilder.AllocParameters<FMyPassParameters>();
PassParams->RenderTargets[0] = FRenderTargetBinding(
    OutputTexture,             // FRDGTextureRef
    ERenderTargetLoadAction::EClear);  // Clear / Load / NoAction

PassParams->RenderTargets.DepthStencil = FDepthStencilBinding(
    DepthTexture,
    ERenderTargetLoadAction::ELoad,
    ERenderTargetLoadAction::ENoAction,
    FExclusiveDepthStencil::DepthRead_StencilNop);
```

---

## FRDGParameterStruct — 構造体の低レベルラッパー

```cpp
class FRDGParameterStruct
{
public:
    bool HasRenderTargets()       const;
    bool HasExternalOutputs()     const;

    uint32 GetBufferParameterCount()       const;
    uint32 GetTextureParameterCount()      const;
    uint32 GetUniformBufferParameterCount() const;

    // パラメータを列挙（RDG 内部でリソース解析に使用）
    template <typename FunctionType>
    void EnumerateTextures(FunctionType Function) const;
    void EnumerateBuffers(FunctionType Function) const;
    void EnumerateUniformBuffers(FunctionType Function) const;
};
```

---

## FRDGBlackboard — パス間データストア

同一フレーム内の異なるパス間でデータを型安全に受け渡すための辞書。  
`FRDGBuilder` に紐付いており `Execute()` 後に破棄される。

```cpp
// Blackboard に格納する構造体を登録（グローバルに一意）
#define RDG_REGISTER_BLACKBOARD_STRUCT(StructType) \
    template <> inline FString FRDGBlackboard::GetTypeName<StructType>() \
    { return TEXT(#StructType); }

// 例: シーンテクスチャを Blackboard で共有する
struct FSceneTextureSharedContext
{
    FRDGTextureRef SceneColor;
    FRDGTextureRef SceneDepth;
    FRDGBufferRef  ExposureBuffer;
};
RDG_REGISTER_BLACKBOARD_STRUCT(FSceneTextureSharedContext)

// ─── パスAで書き込み ───
auto& SharedCtx = GraphBuilder.Blackboard.Create<FSceneTextureSharedContext>();
SharedCtx.SceneColor = SceneColorTexture;
SharedCtx.SceneDepth = SceneDepthTexture;

// ─── パスBで読み込み ───
const auto& SharedCtx = GraphBuilder.Blackboard.GetChecked<FSceneTextureSharedContext>();
FRDGTextureRef Color = SharedCtx.SceneColor;

// ─── 存在チェック付きアクセス ───
const auto* SharedCtx = GraphBuilder.Blackboard.GetMutable<FSceneTextureSharedContext>();
if (SharedCtx) { ... }

// ─── 存在しなければ作成 ───
auto& SharedCtx = GraphBuilder.Blackboard.GetOrCreate<FSceneTextureSharedContext>();
```

---

## FRDGBlackboard API

```cpp
class FRDGBlackboard
{
public:
    // 新規作成（すでに存在する場合はエラー）
    template <typename StructType, typename... ArgsType>
    StructType& Create(ArgsType&&... Args);

    // 存在チェック付き取得（const 参照）
    template <typename StructType>
    const StructType& GetChecked() const;

    // Mutable 取得（nullptr なら存在しない）
    template <typename StructType>
    StructType* GetMutable() const;

    // 存在しなければデフォルト構築して返す
    template <typename StructType, typename... ArgsType>
    StructType& GetOrCreate(ArgsType&&... Args);
};

// FRDGBuilder から直接アクセス
FRDGBlackboard& Blackboard = GraphBuilder.Blackboard;
```

---

## GPU プロファイル / イベントスコープ

GPU プロファイラ（RenderDoc / PIX / Nsight）に表示される階層スコープ。

```cpp
// パス名に変数を埋め込む
GraphBuilder.AddPass(
    RDG_EVENT_NAME("Bloom Downsample %dx%d", Width, Height),
    ...);

// スコープを作る（入れ子にできる）
{
    RDG_EVENT_SCOPE(GraphBuilder, "PostProcess");
    {
        RDG_EVENT_SCOPE(GraphBuilder, "Bloom");
        GraphBuilder.AddPass(RDG_EVENT_NAME("Setup"), ...);
        GraphBuilder.AddPass(RDG_EVENT_NAME("Downsample"), ...);
    }
    GraphBuilder.AddPass(RDG_EVENT_NAME("Tonemap"), ...);
}

// GPU stat スコープ（r.GPU.StatGroup と連動）
RDG_GPU_STAT_SCOPE(GraphBuilder, Bloom);

// CSV プロファイルスコープ
RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, PostProcess);
```

---

## ScreenPass ユーティリティ

PostProcess パスでよく使われる `FScreenPassTexture` / `FScreenPassRenderTarget`:

```cpp
// スクリーンパスのテクスチャラッパー（ビューポート情報込み）
struct FScreenPassTexture
{
    FRDGTextureRef Texture = nullptr;
    FIntRect Viewport;

    bool IsValid() const { return Texture != nullptr; }
};

struct FScreenPassRenderTarget : public FScreenPassTexture
{
    ERenderTargetLoadAction LoadAction = ERenderTargetLoadAction::ENoAction;

    FRenderTargetBinding GetRenderTargetBinding() const;
};

// 典型的な使用パターン（PostProcess パス関数の戻り値）
FScreenPassTexture AddMyPostProcessPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FScreenPassTexture Input);
```

---

## 関連リファレンス

| リファレンス | 対象ソース |
|------------|----------|
| [[ref_rdg_parameter]] | `RenderGraphParameter.h`, `RenderGraphParameters.inl` |
| [[ref_rdg_blackboard]] | `RenderGraphBlackboard.h/.cpp` |
| [[ref_rdg_utils]] | `RenderGraphUtils.h/.cpp` |
| [[ref_rdg_event]] | `RenderGraphEvent.h/.inl/.cpp` |
