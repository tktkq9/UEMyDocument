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

---

## FRDGUserValidation

```cpp
#if RDG_ENABLE_DEBUG

class FRDGUserValidation final
{
public:
    FRDGUserValidation(FRDGAllocator& Allocator);
    ~FRDGUserValidation();

    // ──── リソース作成時の検証 ────
    void ValidateCreateTexture(const FRDGTextureDesc& Desc, const TCHAR* Name, ERDGTextureFlags Flags);
    void ValidateCreateBuffer(const FRDGBufferDesc& Desc, const TCHAR* Name, ERDGBufferFlags Flags);
    void ValidateCreateSRV(const FRDGTextureSRVDesc& Desc);
    void ValidateCreateSRV(const FRDGBufferSRVDesc& Desc);
    void ValidateCreateUAV(const FRDGTextureUAVDesc& Desc);
    void ValidateCreateUAV(const FRDGBufferUAVDesc& Desc);
    void ValidateCreateUniformBuffer(const void* ParameterStruct, const FShaderParametersMetadata* Metadata);

    // ──── 外部リソース登録時の検証 ────
    void ValidateCreateTexture(FRDGTextureRef Texture);
    void ValidateCreateBuffer(FRDGBufferRef Buffer);
    void ValidateCreateSRV(FRDGTextureSRVRef SRV);
    void ValidateCreateSRV(FRDGBufferSRVRef SRV);
    void ValidateCreateUAV(FRDGTextureUAVRef UAV);
    void ValidateCreateUAV(FRDGBufferUAVRef UAV);
    void ValidateCreateUniformBuffer(TRDGUniformBufferRef<void> UniformBuffer);
    void ValidateRegisterExternalTexture(const FRDGTexture* Texture);
    void ValidateRegisterExternalBuffer(const FRDGBuffer* Buffer);

    // ──── パス追加時の検証 ────
    void ValidateAddPass(FRDGPassRef Pass, const FShaderParametersMetadata* ParametersMetadata, const void* ParameterStruct);
    void ValidateExecuteBegin(const TCHAR* BuilderName);

    // ──── 実行時の検証 ────
    void ValidateExecutePassBegin(FRDGPassRef Pass);
    void ValidateExecutePassEnd(FRDGPassRef Pass);
    void ValidateExecuteEnd();

    // ──── グラフ抽出の検証 ────
    void ValidateExtractTexture(FRDGTextureRef Texture, const TRefCountPtr<IPooledRenderTarget>* OutPooledTexture);
    void ValidateExtractBuffer(FRDGBufferRef Buffer, const TRefCountPtr<FRDGPooledBuffer>* OutPooledBuffer);
};

#endif // RDG_ENABLE_DEBUG
```

---

## 主な検証項目

| 検証タイミング | 検証内容 |
|--------------|---------|
| `CreateTexture` / `CreateBuffer` | 名前が null でないか、Desc が有効か |
| `RegisterExternalTexture` | 同じリソースを二重登録していないか |
| `AddPass` | パラメータ構造体のリソースが全てグラフに登録済みか |
| `AddPass` | 書き込みリソースが NeverCull なしで参照されているか |
| `ExecutePassBegin/End` | パス内でのリソースアクセスが宣言と一致するか |
| `Execute` | グラフ全体の参照関係が閉じているか |
| `ExtractTexture` | 抽出するリソースがグラフ内で生産されているか |

---

## CVar による制御

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.RDG.Validation` | 1 | 0 にするとバリデーション全体を無効化（`RDG_ENABLE_DEBUG` が必要） |

---

## 使用箇所

`FRDGBuilder` のメンバ変数として内部保持される:

```cpp
class FRDGBuilder
{
    // ...
#if RDG_ENABLE_DEBUG
    FRDGUserValidation UserValidation;
#endif
};
```

バリデーション API は `FRDGBuilder` の各 `Create` / `AddPass` / `Execute` から自動的に呼ばれる。  
ユーザーコードからは直接使用しない。
