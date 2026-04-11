# リファレンス：RenderGraphUtils.h / RenderGraphUtils.cpp

- グループ: d - Parameters
- 上位: [[d_rdg_parameters]]
- 関連: [[ref_rdg_parameter]] | [[ref_rdg_event]] | [[ref_rdg_builder]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphUtils.h`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphUtils.cpp`

## 概要

レンダリングコードでよく使われる RDG ユーティリティ関数群。  
リソース状態確認・レンダーターゲットバインド・クリア・コピー・スクリーンパスラッパーを含む。

---

## イベント名マクロ

```cpp
// GPU プロファイラに表示されるパス名を生成
// フォーマット文字列あり: RenderDoc/PIX に変数値が表示される
#define RDG_EVENT_NAME(Format, ...) \
    FRDGEventName(TEXT(Format), ##__VA_ARGS__)

// 注意: フォーマット引数なし版は静的文字列として最適化される
RDG_EVENT_NAME("StaticPassName")         // 高速（文字列コピーなし）
RDG_EVENT_NAME("Pass %dx%d", W, H)       // 動的（文字列フォーマット）
```

---

## イベントスコープマクロ

```cpp
// RDG グラフ全体のスコープ（GPU プロファイラで階層化される）
RDG_EVENT_SCOPE(GraphBuilder, "MySubsystem");
RDG_EVENT_SCOPE(GraphBuilder, "Bloom Pass %s", bFFT ? "FFT" : "Gaussian");

// GPU 統計スコープ（r.GPU.Stat.MyStatName で計測）
RDG_GPU_STAT_SCOPE(GraphBuilder, MyStatName);

// CSV プロファイルスコープ
RDG_CSV_STAT_EXCLUSIVE_SCOPE(GraphBuilder, PostProcess);
```

---

## FRDGEventName

```cpp
class FRDGEventName final
{
public:
    FRDGEventName(const TCHAR* EventFormat, ...);  // 可変長引数（動的）
    FRDGEventName(int32 NonVariadic, const TCHAR* EventName);  // 静的文字列

    const TCHAR* GetTCHAR() const;
    bool HasFormattedString() const;
};
```

---

## FRDGBlackboard

```cpp
class FRDGBlackboard
{
public:
    // 新規作成（同一型はひとつまで。2回呼ぶとエラー）
    template <typename StructType, typename... ArgsType>
    StructType& Create(ArgsType&&... Args);

    // 存在チェック付き取得（const 参照、なければアサート）
    template <typename StructType>
    const StructType& GetChecked() const;

    // Mutable 取得（存在しなければ nullptr）
    template <typename StructType>
    StructType* GetMutable() const;

    // 存在しなければデフォルト構築して返す
    template <typename StructType, typename... ArgsType>
    StructType& GetOrCreate(ArgsType&&... Args);
};

// 使用登録マクロ（cpp ファイルで1回だけ呼ぶ）
#define RDG_REGISTER_BLACKBOARD_STRUCT(StructType)
```

---

## FRDGParameterStruct

```cpp
class FRDGParameterStruct
{
public:
    template <typename ParameterStructType>
    explicit FRDGParameterStruct(
        const ParameterStructType* Parameters,
        const FShaderParametersMetadata* InParameterMetadata);

    const uint8*                    GetContents() const;
    const FRHIUniformBufferLayout&  GetLayout()   const;
    const FShaderParametersMetadata* GetMetadata() const;

    bool HasRenderTargets()        const;
    bool HasExternalOutputs()      const;
    bool HasTextures()             const;
    bool HasBuffers()              const;
    bool HasUniformBuffers()       const;

    uint32 GetBufferParameterCount()        const;
    uint32 GetTextureParameterCount()       const;
    uint32 GetUniformBufferParameterCount() const;

    // リソース列挙（RDG 内部でバリア解析に使用）
    template <typename FunctionType>
    void Enumerate(FunctionType Function) const;
    void EnumerateTextures(FunctionType Function) const;
    void EnumerateBuffers(FunctionType Function) const;
    void EnumerateUniformBuffers(FunctionType Function) const;
};
```

---

## シェーダーパラメータ構造体マクロ一覧

```cpp
// 構造体の開始・終了
BEGIN_SHADER_PARAMETER_STRUCT(StructName, Qualifier)
END_SHADER_PARAMETER_STRUCT()

// プリミティブ値
SHADER_PARAMETER(Type, Name)                    // float, uint32, FVector2f 等
SHADER_PARAMETER_ARRAY(Type, Name, [Count])

// RDG テクスチャ
SHADER_PARAMETER_RDG_TEXTURE(TextureType, Name)            // SRV として使用
SHADER_PARAMETER_RDG_TEXTURE_SRV(TextureType, Name)        // 明示的 SRV
SHADER_PARAMETER_RDG_TEXTURE_UAV(TextureType, Name)        // UAV
SHADER_PARAMETER_RDG_TEXTURE_ARRAY(TextureType, Name, [N]) // 配列

// RDG バッファ
SHADER_PARAMETER_RDG_BUFFER_SRV(BufferType, Name)
SHADER_PARAMETER_RDG_BUFFER_UAV(BufferType, Name)

// Uniform Buffer
SHADER_PARAMETER_STRUCT_REF(StructType, Name)              // TUniformBufferRef<>
SHADER_PARAMETER_RDG_UNIFORM_BUFFER(StructType, Name)      // TRDGUniformBufferRef<>
SHADER_PARAMETER_STRUCT(StructType, Name)                  // インライン埋め込み
SHADER_PARAMETER_STRUCT_ARRAY(StructType, Name, [N])

// サンプラー / テクスチャ（非 RDG。StaticSamplerState 等）
SHADER_PARAMETER_SAMPLER(SamplerType, Name)
SHADER_PARAMETER_TEXTURE(TextureType, Name)

// レンダーターゲット（Raster パスのみ）
RENDER_TARGET_BINDING_SLOTS()
```

---

## RenderGraphUtils.h — ユーティリティ関数

```cpp
// テクスチャをクリア（RDG パスとして追加）
void AddClearRenderTargetPass(
    FRDGBuilder& GraphBuilder,
    FRDGTextureRef Texture,
    const FLinearColor& ClearColor = FLinearColor::Black);

void AddClearDepthStencilPass(
    FRDGBuilder& GraphBuilder,
    FRDGTextureRef Texture,
    bool bClearDepth, float Depth,
    bool bClearStencil, uint8 Stencil);

// テクスチャ → テクスチャコピー
void AddCopyTexturePass(
    FRDGBuilder& GraphBuilder,
    FRDGTextureRef InputTexture,
    FRDGTextureRef OutputTexture,
    const FRHICopyTextureInfo& CopyInfo = FRHICopyTextureInfo());

// バッファをゼロクリア
void AddClearUAVPass(
    FRDGBuilder& GraphBuilder,
    FRDGBufferUAVRef BufferUAV,
    uint32 ClearValue);

void AddClearUAVFloatPass(
    FRDGBuilder& GraphBuilder,
    FRDGBufferUAVRef BufferUAV,
    float ClearValue);

// GPU → CPU リードバック（非同期）
void AddEnqueueCopyPass(
    FRDGBuilder& GraphBuilder,
    FRHIGPUTextureReadback* Readback,
    FRDGTextureRef SourceTexture,
    FResolveParams ResolveParams = FResolveParams());

void AddEnqueueCopyPass(
    FRDGBuilder& GraphBuilder,
    FRHIGPUBufferReadback* Readback,
    FRDGBufferRef SourceBuffer,
    uint32 NumBytes = 0);
```

---

## FComputeShaderUtils — Compute パスのショートカット

```cpp
// Dispatch（グループ数指定）
template<typename TShaderClass>
static void Dispatch(
    FRHIComputeCommandList& RHICmdList,
    const TShaderRef<TShaderClass>& Shader,
    const typename TShaderClass::FParameters& Parameters,
    FIntVector GroupCount);

// Dispatch（スレッド数から自動計算）
template<typename TShaderClass>
static void DispatchIndirect(
    FRHIComputeCommandList& RHICmdList,
    const TShaderRef<TShaderClass>& Shader,
    const typename TShaderClass::FParameters& Parameters,
    FRHIBuffer* IndirectArgsBuffer, uint32 IndirectArgOffset);

// AddPass のショートカット（GraphBuilder 版）
template<typename TShaderClass>
static FRDGPassRef AddPass(
    FRDGBuilder& GraphBuilder,
    FRDGEventName&& PassName,
    const TShaderRef<TShaderClass>& Shader,
    typename TShaderClass::FParameters* Parameters,
    FIntVector GroupCount);

// グループ数の計算（スレッド総数 / グループサイズ、端数切り上げ）
static FIntVector GetGroupCount(const FIntPoint& ThreadCount, const FIntPoint& GroupSize);
static FIntVector GetGroupCount(const FIntVector& ThreadCount, const FIntVector& GroupSize);
static int32 GetGroupCount(int32 ThreadCount, int32 GroupSize);
```
