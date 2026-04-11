# リファレンス：RenderGraphDefinitions.h / RenderGraphFwd.h

- グループ: a - Builder
- 上位: [[a_rdg_builder]]
- 関連: [[ref_rdg_builder]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphDefinitions.h`
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphFwd.h`

## 概要

RDG 全体で使われるマクロ定数・コンパイル条件・前方宣言を定義するヘッダ。  
`RenderGraph.h` がアンブレラインクルードとしてすべての RDG ヘッダをまとめる。

---

## コンパイル条件マクロ

### RDG_ENABLE_DEBUG

```cpp
#define RDG_ENABLE_DEBUG (!UE_BUILD_SHIPPING && !UE_BUILD_TEST)
```

Shipping / Test ビルドでデバッグ機能（バリデーション・即時モード CVar 等）を無効化する。  
このマクロが `0` の場合、`FRDGUserValidation` の全メソッドが空実装になる。

### 使用箇所
- `RenderGraphBuilder.h` の `UserValidation` メンバの `#if RDG_ENABLE_DEBUG` ガード
- [[ref_rdg_validation]] の `FRDGUserValidation` クラス全体

---

### RDG_ENABLE_TRACE

```cpp
#define RDG_ENABLE_TRACE UE_TRACE_ENABLED && !IS_PROGRAM && !UE_BUILD_SHIPPING
```

Unreal Insights へのトレース出力を制御する。

### 使用箇所
- [[ref_rdg_trace]] の `FRDGTrace` クラス全体

---

### RDG_EVENTS（GPU イベントレベル）

```cpp
#define RDG_EVENTS_NONE       0  // GPU イベント名を一切評価しない
#define RDG_EVENTS_STRING_REF 1  // const TCHAR* のみ保持
#define RDG_EVENTS_STRING_COPY 2 // FString に展開して保持
```

ビルド構成による `RDG_EVENTS` の実際の値:

| ビルド構成 | `RDG_EVENTS` |
|-----------|-------------|
| Shipping / Test（WITH_PROFILEGPU） | `STRING_REF` |
| Debug / Development | `STRING_COPY` |
| WITH_RHI_BREADCRUMBS のみ | `STRING_REF` |
| その他 | `NONE` |

### 使用箇所
- [[ref_rdg_event]] の `FRDGEventName` クラス内部

---

### その他のマクロ

| マクロ | 説明 |
|-------|------|
| `IF_RDG_ENABLE_DEBUG(Op)` | `RDG_ENABLE_DEBUG` が有効な場合のみ `Op` を実行 |
| `IF_RDG_ENABLE_TRACE(Op)` | `RDG_ENABLE_TRACE` が有効な場合のみ `Op` を実行 |
| `RDG_DUMP_RESOURCES` | `WITH_DUMPGPU` — GPU ダンプ機能 |
| `SUPPORTS_VISUALIZE_TEXTURE` | テクスチャビジュアライザの有効条件 |

---

## 並列実行の注意事項（定義コメントより）

`FRHICommandList&` を引数に取るパスは `ERDGBuilderFlags::ParallelExecute` 時に並列実行される。  
並列実行されたタスクは `Execute()` 終了時にデフォルトで待機される。  
`FRDGAsyncTask` をタグとしてラムダの第一引数に付けた場合は **待機されない** — 手動で `WaitForAsyncExecuteTask()` を呼ぶ必要がある。

```cpp
// 例: 非同期タスクとして実行されるパス（Execute() 後も走り続ける）
GraphBuilder.AddPass(RDG_EVENT_NAME("AsyncPass"), PassParams, ERDGPassFlags::Compute,
    [](FRDGAsyncTask, FRHICommandList& RHICmdList)
    {
        // Execute() 後も非同期で実行される
    });

// 後で手動待機
FRDGBuilder::WaitForAsyncExecuteTask();
```

---

## RenderGraphFwd.h — 前方宣言・型エイリアス一覧

```cpp
// クラス前方宣言
class FRDGBuilder;
class FRDGPass;
class FRDGResource;
class FRDGTexture;
class FRDGBuffer;
class FRDGShaderResourceView;
class FRDGUnorderedAccessView;
class FRDGUniformBuffer;
class FRDGPooledBuffer;
class FRDGPooledTexture;

// ポインタ型エイリアス（"Ref" = 生ポインタ）
using FRDGPassRef                 = FRDGPass*;
using FRDGResourceRef             = FRDGResource*;
using FRDGTextureRef              = FRDGTexture*;
using FRDGBufferRef               = FRDGBuffer*;
using FRDGTextureSRVRef           = FRDGTextureSRV*;
using FRDGBufferSRVRef            = FRDGBufferSRV*;
using FRDGTextureUAVRef           = FRDGTextureUAV*;
using FRDGBufferUAVRef            = FRDGBufferUAV*;
using FRDGUniformBufferRef        = FRDGUniformBuffer*;
using FRDGShaderResourceViewRef   = FRDGShaderResourceView*;
using FRDGUnorderedAccessViewRef  = FRDGUnorderedAccessView*;
using FRDGViewRef                 = FRDGView*;

// 型付き Uniform Buffer
template <typename TUniformStruct>
using TRDGUniformBufferRef = TRDGUniformBuffer<TUniformStruct>*;
```

> **注意**: `FRDGTextureSRV` / `FRDGBufferUAV` などは `FRDGShaderResourceView` / `FRDGUnorderedAccessView` の  
> 型エイリアスではなく **別の具体型**。`FRDGTextureSRVRef` と `FRDGBufferSRVRef` は  
> ともに `FRDGShaderResourceView*` だが、前方宣言上は別名で宣言されている。

---

## アンブレラインクルード（RenderGraph.h）

```cpp
// RenderGraph.h が以下を一括 include する
#include "RenderGraphBuilder.h"
#include "RenderGraphResources.h"
#include "RenderGraphPass.h"
#include "RenderGraphUtils.h"
#include "RenderGraphBlackboard.h"
#include "RenderGraphEvent.h"
#include "RenderGraphParameter.h"
```

通常のレンダリングコードは `#include "RenderGraph.h"` 一行で RDG 全体を使用できる。

### 使用箇所
- `DeferredShadingRenderer.h` — レンダラー全体の include
- `ScreenPass.h` — ポストプロセス系の include
