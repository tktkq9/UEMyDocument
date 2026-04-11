# リファレンス：RenderGraphBuilder.h / RenderGraphBuilder.inl / RenderGraphBuilder.cpp

- グループ: a - Builder
- 上位: [[a_rdg_builder]]
- 関連: [[ref_rdg_definitions]] | [[ref_rdg_pass]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphBuilder.h`
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphBuilder.inl`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphBuilder.cpp`

## 概要

レンダーグラフ全体を管理するメインクラス `FRDGBuilder` の全 API 定義。  
グラフの構築・コンパイル・実行を担う。

---

## コンストラクタ / デストラクタ

```cpp
FRDGBuilder(
    FRHICommandListImmediate& RHICmdList,
    FRDGEventName Name = {},
    ERDGBuilderFlags Flags = ERDGBuilderFlags::None,
    EShaderPlatform ShaderPlatform = GMaxRHIShaderPlatform);

~FRDGBuilder();  // Execute() 前に破棄すると警告
```

---

## リソース検索

```cpp
FRDGTexture* FindExternalTexture(FRHITexture* Texture) const;
FRDGTexture* FindExternalTexture(IPooledRenderTarget* ExternalPooledTexture) const;
FRDGBuffer*  FindExternalBuffer(FRHIBuffer* Buffer) const;
FRDGBuffer*  FindExternalBuffer(FRDGPooledBuffer* ExternalPooledBuffer) const;
```

---

## 外部リソース登録（RegisterExternal）

```cpp
// テクスチャ登録
FRDGTextureRef RegisterExternalTexture(
    const TRefCountPtr<IPooledRenderTarget>& ExternalPooledTexture,
    ERDGTextureFlags Flags = ERDGTextureFlags::None);

FRDGTextureRef RegisterExternalTexture(
    const TRefCountPtr<IPooledRenderTarget>& ExternalPooledTexture,
    const TCHAR* NameIfNotRegistered,
    ERDGTextureFlags Flags = ERDGTextureFlags::None);

// バッファ登録
FRDGBufferRef RegisterExternalBuffer(
    const TRefCountPtr<FRDGPooledBuffer>& ExternalPooledBuffer,
    ERDGBufferFlags Flags = ERDGBufferFlags::None);

FRDGBufferRef RegisterExternalBuffer(
    const TRefCountPtr<FRDGPooledBuffer>& ExternalPooledBuffer,
    ERDGBufferFlags Flags,
    ERHIAccess AccessFinal);

FRDGBufferRef RegisterExternalBuffer(
    const TRefCountPtr<FRDGPooledBuffer>& ExternalPooledBuffer,
    const TCHAR* NameIfNotRegistered,
    ERDGBufferFlags Flags = ERDGBufferFlags::None);
```

---

## リソース作成（Create）

```cpp
// テクスチャ
FRDGTextureRef CreateTexture(
    const FRDGTextureDesc& Desc,
    const TCHAR* Name,
    ERDGTextureFlags Flags = ERDGTextureFlags::None);

// バッファ（要素数固定）
FRDGBufferRef CreateBuffer(
    const FRDGBufferDesc& Desc,
    const TCHAR* Name,
    ERDGBufferFlags Flags = ERDGBufferFlags::None);

// バッファ（要素数コールバック）
FRDGBufferRef CreateBuffer(
    const FRDGBufferDesc& Desc,
    const TCHAR* Name,
    FRDGBufferNumElementsCallback&& NumElementsCallback,
    ERDGBufferFlags Flags = ERDGBufferFlags::None);
```

---

## View 作成（CreateSRV / CreateUAV）

```cpp
// SRV
FRDGTextureSRVRef CreateSRV(const FRDGTextureSRVDesc& Desc);
FRDGBufferSRVRef  CreateSRV(const FRDGBufferSRVDesc& Desc);
FRDGBufferSRVRef  CreateSRV(FRDGBufferRef Buffer, EPixelFormat Format);  // ショートカット

// UAV
FRDGTextureUAVRef CreateUAV(const FRDGTextureUAVDesc& Desc,
    ERDGUnorderedAccessViewFlags Flags = ERDGUnorderedAccessViewFlags::None);
FRDGTextureUAVRef CreateUAV(FRDGTextureRef Texture,
    ERDGUnorderedAccessViewFlags Flags = ERDGUnorderedAccessViewFlags::None,
    EPixelFormat Format = PF_Unknown);
FRDGBufferUAVRef CreateUAV(const FRDGBufferUAVDesc& Desc,
    ERDGUnorderedAccessViewFlags Flags = ERDGUnorderedAccessViewFlags::None);
FRDGBufferUAVRef CreateUAV(FRDGBufferRef Buffer, EPixelFormat Format,
    ERDGUnorderedAccessViewFlags Flags = ERDGUnorderedAccessViewFlags::None);

// Uniform Buffer（型付き）
template <typename ParameterStructType>
TRDGUniformBufferRef<ParameterStructType> CreateUniformBuffer(
    const ParameterStructType* ParameterStruct);
```

---

## メモリアロケーション

```cpp
void*   Alloc(uint64 SizeInBytes, uint32 AlignInBytes = 16);

template <typename PODType>
PODType* AllocPOD();

template <typename PODType>
PODType* AllocPODArray(uint32 Count);

template <typename PODType>
TArrayView<PODType> AllocPODArrayView(uint32 Count);

template <typename ObjectType, typename... TArgs>
ObjectType* AllocObject(TArgs&&... Args);

template <typename ObjectType>
TArray<ObjectType, SceneRenderingAllocator>& AllocArray();

template <typename ParameterStructType>
ParameterStructType* AllocParameters();

template <typename ParameterStructType>
ParameterStructType* AllocParameters(const ParameterStructType* StructToCopy);

template <typename BaseParameterStructType>
BaseParameterStructType* AllocParameters(const FShaderParametersMetadata* Metadata);

template <typename BaseParameterStructType>
TStridedView<BaseParameterStructType> AllocParameters(
    const FShaderParametersMetadata* Metadata, uint32 NumStructs);
```

---

## パス追加（AddPass）

```cpp
// 型付きパラメータ構造体版
template <typename ParameterStructType, typename ExecuteLambdaType>
FRDGPassRef AddPass(
    FRDGEventName&& Name,
    const ParameterStructType* ParameterStruct,
    ERDGPassFlags Flags,
    ExecuteLambdaType&& ExecuteLambda);

// データドリブンパラメータ版
template <typename ExecuteLambdaType>
FRDGPassRef AddPass(
    FRDGEventName&& Name,
    const FShaderParametersMetadata* ParametersMetadata,
    const void* ParameterStruct,
    ERDGPassFlags Flags,
    ExecuteLambdaType&& ExecuteLambda);

// パラメータなし版（デバッグ用）
template <typename ExecuteLambdaType>
FRDGPassRef AddPass(
    FRDGEventName&& Name,
    ERDGPassFlags Flags,
    ExecuteLambdaType&& ExecuteLambda);

// 並列コマンドリスト版
template <typename ParameterStructType, typename LaunchLambdaType>
FRDGPassRef AddDispatchPass(
    FRDGEventName&& Name,
    const ParameterStructType* ParameterStruct,
    ERDGPassFlags Flags,
    LaunchLambdaType&& LaunchLambda);
```

---

## パス制御

```cpp
void SetPassWorkload(FRDGPass* Pass, uint32 Workload);
void AddPassDependency(FRDGPass* Producer, FRDGPass* Consumer);
void AddDispatchHint();  // RHI スレッドへのフラッシュヒント
```

---

## セットアップタスク（並列前処理）

```cpp
template <typename TaskLambda>
UE::Tasks::FTask AddSetupTask(
    TaskLambda&& Task,
    bool bCondition = true,
    ERDGSetupTaskWaitPoint WaitPoint = ERDGSetupTaskWaitPoint::Compile);

// オーバーロード（Priority / Pipe / Prerequisites の各組み合わせ）
// AddCommandListSetupTask も同様のオーバーロードあり

bool IsParallelSetupEnabled() const;
bool IsAsyncComputeEnabled() const;
```

---

## リソース抽出・クリーンアップ

```cpp
// Execute 後にリソースを外部に戻す
void QueueTextureExtraction(
    FRDGTextureRef Texture,
    TRefCountPtr<IPooledRenderTarget>* OutPooledTexture,
    ERHIAccess AccessFinal = ERHIAccess::Unknown);

void QueueBufferExtraction(
    FRDGBufferRef Buffer,
    TRefCountPtr<FRDGPooledBuffer>* OutPooledBuffer,
    ERHIAccess AccessFinal = ERHIAccess::Unknown);

// 未使用 RHI リソースの解放
void FlushRenderPassAllocations();
```

---

## コールバック・その他

```cpp
// Execute 完了後に呼ばれるコールバック登録
void AddPostExecuteCallback(TUniqueFunction<void()>&& Callback);

// グラフの実行（バリア挿入・最適化・コマンド発行を行う）
void Execute();

// メンバ変数（公開）
FRDGBlackboard Blackboard;  // パス間データストア
```

---

## ERDGBuilderFlags

```cpp
enum class ERDGBuilderFlags {
    None            = 0,
    ParallelSetup   = 1 << 0,
    ParallelCompile = 1 << 1,
    ParallelExecute = 1 << 2,
    Parallel = ParallelSetup | ParallelCompile | ParallelExecute,
};
```
