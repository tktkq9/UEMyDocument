# リファレンス：RenderGraphValidation.h / RenderGraphValidation.cpp

- グループ: e - Debug
- 上位: [[e_rdg_debug_trace]]
- 関連: [[ref_rdg_trace]] | [[ref_rdg_private]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphValidation.h`
- ソース: `Engine/Source/Runtime/RenderCore/Private/RenderGraphValidation.cpp`

## 概要

`FRDGBuilder` の API 使用を検証するバリデーション層。  
`RDG_ENABLE_DEBUG`（非 Shipping / 非 Test）でのみコンパイルされる。  
ユーザー起因の誤用をできる限り早期（セットアップ段階）に検出する。  
**内部グラフ状態の検証はこのクラスに含めない**（外部 API 経由の誤用のみを対象とする）。

---

## FRDGUserValidation

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `Allocator` | `FRDGAllocator&` | リソースマップの確保に使用 |
| `ResourceMap` | `TSet<FRDGResourceRef, ..., FRDGSetAllocator>` | グラフに登録済みのリソース全件 |
| `bHasExecuted` | `bool` | `Execute()` が完了したか |
| `bHasExecuteBegun` | `bool` | `Execute()` が開始したか |
| `bParallelExecuteEnabled` | `bool` | 並列実行モード中か（アクセスチェックの挙動を変える） |

### 公開メソッド

```cpp
// ──── リソース作成時の検証 ────
void ValidateCreateTexture(const FRDGTextureDesc& Desc, const TCHAR* Name, ERDGTextureFlags Flags);
void ValidateCreateBuffer(const FRDGBufferDesc& Desc, const TCHAR* Name, ERDGBufferFlags Flags);
void ValidateCreateSRV(const FRDGTextureSRVDesc& Desc);
void ValidateCreateSRV(const FRDGBufferSRVDesc& Desc);
void ValidateCreateUAV(const FRDGTextureUAVDesc& Desc);
void ValidateCreateUAV(const FRDGBufferUAVDesc& Desc);
void ValidateCreateUniformBuffer(const void* ParameterStruct, const FShaderParametersMetadata* Metadata);

// ──── 外部リソース登録時の検証 ────
void ValidateRegisterExternalTexture(
    const TRefCountPtr<IPooledRenderTarget>& ExternalPooledTexture,
    const TCHAR* Name, ERDGTextureFlags Flags);
void ValidateRegisterExternalBuffer(
    const TRefCountPtr<FRDGPooledBuffer>& ExternalPooledBuffer,
    const TCHAR* Name, ERDGBufferFlags Flags);

// ──── バッファアップロード検証 ────
void ValidateUploadBuffer(FRDGBufferRef Buffer, const void* InitialData, uint64 InitialDataSize);
void ValidateUploadBuffer(FRDGBufferRef Buffer, const FRDGBufferInitialDataFillCallback& Callback);

// ──── パス追加時の検証 ────
// パラメータ構造体ありパスの追加
void ValidateAddPass(const void* ParameterStruct, const FShaderParametersMetadata* Metadata,
                     const FRDGEventName& Name, ERDGPassFlags Flags);
// パラメータなしパスの追加
void ValidateAddPass(const FRDGEventName& Name, ERDGPassFlags Flags);
// AddPass 後の最終検証
void ValidateAddPass(const FRDGPass* Pass);

// ──── 実行時の検証 ────
void ValidateExecuteBegin();
void ValidateExecutePassBegin(const FRDGPass* Pass);
void ValidateExecutePassEnd(const FRDGPass* Pass);
void ValidateExecuteEnd();

// ──── 抽出・変換の検証 ────
void ValidateExtractTexture(FRDGTextureRef Texture, TRefCountPtr<IPooledRenderTarget>* OutTexturePtr);
void ValidateExtractBuffer(FRDGBufferRef Buffer, TRefCountPtr<FRDGPooledBuffer>* OutBufferPtr);
void ValidateConvertToExternalResource(FRDGViewableResource* Resource);
void ValidateConvertToExternalUniformBuffer(FRDGUniformBuffer* UniformBuffer);

// ──── アクセスモード検証 ────
void ValidateUseExternalAccessMode(FRDGViewableResource* Resource,
                                   ERHIAccess ReadOnlyAccess, ERHIPipeline Pipelines);
void ValidateUseInternalAccessMode(FRDGViewableResource* Resource);
void ValidateExternalAccess(FRDGViewableResource* Resource, ERHIAccess Access, const FRDGPass* Pass);

// ──── その他 ────
bool TryMarkForClobber(FRDGViewableResource* Resource) const;
void ValidateGetPooledTexture(FRDGTextureRef Texture) const;
void ValidateGetPooledBuffer(FRDGBufferRef Buffer) const;
void ValidateSetAccessFinal(FRDGViewableResource* Resource, ERHIAccess AccessFinal);
void SetParallelExecuteEnabled(bool bInParallelExecuteEnabled);

// パスラムダ前後に全リソースの bAllowRHIAccess を切り替える（静的関数）
static void SetAllowRHIAccess(const FRDGPass* Pass, bool bAllowAccess);
```

---

## 主な検証項目

| 検証タイミング | 検証内容 |
|--------------|---------|
| `CreateTexture` / `CreateBuffer` | 名前が null でないか、Desc が有効か |
| `RegisterExternalTexture` | 同じリソースを二重登録していないか |
| `AddPass`（構造体あり） | パラメータ内の全 RDG リソースがグラフに登録済みか |
| `AddPass`（構造体あり） | 書き込みリソースが NeverCull なしで参照されているか |
| `AddPass` | Async / Await パスが Inline 引数を使っていないか |
| `ExecutePassBegin` | パスが宣言したアクセス権限（読み込み / 書き込み）と実際のアクセスが一致するか |
| `ExecutePassEnd` | パスラムダ終了後に RHI Access フラグを false に戻す |
| `ExecuteEnd` | グラフ全体の参照関係が閉じているか（全リソースが消費済みか） |
| `ExtractTexture` | 抽出リソースがグラフ内で生産されているか |
| `ConvertToExternalResource` | 変換対象が正しい状態にあるか |
| `UseExternalAccessMode` | 外部公開するアクセスが ReadOnly であるか（ReadWrite は不可） |

---

## FRDGBarrierValidation

バリアバッチの正当性を検証するクラス（実行フェーズ）。

### メンバ変数

| 変数 | 型 | 説明 |
|-----|-----|------|
| `BatchMap` | `TMap<const FRDGBarrierBatchBegin*, FResourceMap>` | Begin バッチとリソース遷移情報のマップ |
| `Passes` | `const FRDGPassRegistry*` | 全パスの登録情報 |
| `GraphName` | `const TCHAR*` | デバッグ表示用のグラフ名 |

```cpp
class FRDGBarrierValidation
{
public:
    FRDGBarrierValidation(const FRDGPassRegistry* InPasses, const FRDGEventName& InGraphName);

    // Begin バッチを Submit 直前に検証
    void ValidateBarrierBatchBegin(const FRDGPass* Pass, const FRDGBarrierBatchBegin& Batch);

    // End バッチを Submit 直前に検証
    void ValidateBarrierBatchEnd(const FRDGPass* Pass, const FRDGBarrierBatchEnd& Batch);
};
```

---

## FRDGBuilder との連携

```cpp
class FRDGBuilder
{
#if RDG_ENABLE_DEBUG
    FRDGUserValidation UserValidation;  // セットアップ〜実行を通じて常駐
#endif
};
```

バリデーション API は `FRDGBuilder` の `Create*` / `RegisterExternal*` / `AddPass` / `Execute` から  
`IF_RDG_ENABLE_DEBUG(UserValidation.Validate...())` マクロ経由で自動呼び出しされる。  
ユーザーコードから直接使用する必要はない。

---

> [!note]- ExecuteGuard — Execute 後の二重操作を防ぐ
> ```cpp
> void ExecuteGuard(const TCHAR* Operation, const TCHAR* ResourceName);
> ```
> `Execute()` 完了後にリソース操作（`CreateTexture` 等）が呼ばれると  
> `ExecuteGuard` がアサートし、「グラフ実行後のグラフ変更」を検出する。  
> `bHasExecuted` フラグで判定される。

> [!note]- SetAllowRHIAccess の役割
> パスラムダ実行の前後で、パスが参照するリソースの `bAllowRHIAccess` を切り替える静的メソッド。  
> ラムダ実行中（= `GRDGAllowRHIAccess = true` の間）のみ `GetRHI()` が呼べる。  
> これによりパスラムダ外での RHI アクセスを検出できる。

> [!note]- ValidateAddSubresourceAccess — 詳細アクセス検証
> ```cpp
> void ValidateAddSubresourceAccess(FRDGViewableResource* Resource,
>     const FRDGSubresourceState& Subresource, ERHIAccess Access);
> ```
> パスがサブリソース単位で宣言したアクセス状態が、リソースの現在状態と整合しているかを検証する。  
> テクスチャの特定 Mip のみ UAV として書き込む場合等の細粒度チェックに使用。
