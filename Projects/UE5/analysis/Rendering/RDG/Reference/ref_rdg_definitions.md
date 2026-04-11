# リファレンス：RenderGraphDefinitions.h / RenderGraphFwd.h

- グループ: a - Builder
- 上位: [[a_rdg_builder]]
- 関連: [[ref_rdg_builder]]
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphDefinitions.h`
- ソース: `Engine/Source/Runtime/RenderCore/Public/RenderGraphFwd.h`

## 概要

RDG 全体で使われるマクロ定数・コンパイル条件・前方宣言を定義するヘッダ。  
`RenderGraph.h` からすべての RDG ヘッダが一括 include される。

---

## コンパイル条件マクロ

| マクロ | 値 / 条件 | 説明 |
|-------|-----------|------|
| `RDG_ENABLE_DEBUG` | `!UE_BUILD_SHIPPING && !UE_BUILD_TEST` | Shipping / Test ビルドでデバッグ機能を無効化 |
| `RDG_ENABLE_DEBUG_WITH_ENGINE` | `RDG_ENABLE_DEBUG && WITH_ENGINE` | エンジン同梱時のみ有効なデバッグ |
| `RDG_ENABLE_TRACE` | `UE_TRACE_ENABLED && !IS_PROGRAM && !UE_BUILD_SHIPPING` | Unreal Insights トレース出力の有効化 |
| `RDG_DUMP_RESOURCES` | `WITH_DUMPGPU` | GPU ダンプ機能の有効化 |
| `SUPPORTS_VISUALIZE_TEXTURE` | `WITH_ENGINE && !UE_BUILD_SHIPPING` | テクスチャビジュアライザの有効化 |

---

## GPU イベントレベルマクロ

```cpp
#define RDG_EVENTS_NONE       0  // GPU イベント名を完全に無効化
#define RDG_EVENTS_STRING_REF 1  // フォーマット文字列を const TCHAR* として保持
#define RDG_EVENTS_STRING_COPY 2 // フォーマット文字列を FString に展開・保持
```

`RDG_EVENTS` の実際の値はビルド構成によって決定される:

| ビルド構成 | `RDG_EVENTS` 値 |
|-----------|----------------|
| Test / Shipping（WITH_PROFILEGPU） | `RDG_EVENTS_STRING_REF` |
| Debug / Development | `RDG_EVENTS_STRING_COPY` |
| WITH_RHI_BREADCRUMBS のみ | `RDG_EVENTS_STRING_REF` |
| その他 | `RDG_EVENTS_NONE` |

---

## デバッグ補助マクロ

```cpp
// RDG_ENABLE_DEBUG が有効な場合のみ Op を実行
IF_RDG_ENABLE_DEBUG(Op)

// RDG_ENABLE_TRACE が有効な場合のみ Op を実行
IF_RDG_ENABLE_TRACE(Op)
```

---

## 並列実行モード定数

```cpp
/** パス実行ラムダの並列実行モード。
 *  FRHICommandList& を引数に取るパスは FRDGBuilderFlags::ParallelExecute 時に並列実行される。
 *  FRHICommandListImmediate& を取るパスは常にインライン実行される。
 *  FRDGAsyncTask をタグ付けしたラムダは Execute() 終了時に待機されず、
 *  FRDGBuilder::WaitForAsyncExecuteTasks() で手動待機が必要。
 */
```

---

## RenderGraphFwd.h — 主要前方宣言

```cpp
// クラス前方宣言
class FRDGBuilder;
class FRDGPass;
class FRDGResource;
class FRDGViewableResource;
class FRDGTexture;
class FRDGBuffer;
class FRDGView;
class FRDGShaderResourceView;
class FRDGUnorderedAccessView;
class FRDGUniformBuffer;
class FRDGBlackboard;
class FRDGEventName;
class FRDGParameterStruct;

// ハンドル前方宣言
struct FRDGTextureHandle;
struct FRDGBufferHandle;
struct FRDGPassHandle;
```

---

## アンブレラインクルード

```cpp
// RenderGraph.h — すべての RDG ヘッダを一括 include
#include "RenderGraphBuilder.h"
#include "RenderGraphResources.h"
#include "RenderGraphPass.h"
#include "RenderGraphUtils.h"
#include "RenderGraphBlackboard.h"
#include "RenderGraphEvent.h"
#include "RenderGraphParameter.h"
```

通常のレンダリングコードは `#include "RenderGraph.h"` 一行で RDG 全体を使用できる。
