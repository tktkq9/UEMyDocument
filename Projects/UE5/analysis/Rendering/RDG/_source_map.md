# RDG ソースマップ

- 対象: Render Dependency Graph（`FRDGBuilder` 中心）
- 更新日: 2026-04-19
- 上位: [[Rendering/_source_map]]
- 概要: [[10_rdg_overview]]

RDG は RenderCore モジュールが提供する宣言的レンダーグラフ。
全レンダリングパスが依存する基盤層。

---

## ソースパス

| 対象 | パス |
|------|------|
| RDG 公開ヘッダ | `Engine/Source/Runtime/RenderCore/Public/` |
| RDG 実装 | `Engine/Source/Runtime/RenderCore/Private/` |
| RDG 起点 | `Engine/Source/Runtime/Renderer/Private/SceneRenderBuilder.cpp` |

---

## ファイル → クラス対応（公開ヘッダ）

### Builder（グラフ構築・実行）

| ファイル | 主要クラス / 関数 | 役割 | 参照 |
|---------|----------------|------|------|
| `Public/RenderGraphBuilder.h` | `FRDGBuilder` | グラフ構築・Execute | [[Reference/ref_rdg_builder]] |
| `Public/RenderGraphBuilder.inl` | インライン実装 | CreateTexture/Buffer、AllocParameters | 同上 |
| `Private/RenderGraphBuilder.cpp` | `FRDGBuilder::Execute()`:1755, `Compile()`, `Collect()` | コンパイル・バリア挿入・実行 | [[Details/a_rdg_builder]] |
| `Public/RenderGraph.h` | 一括 include | — | — |

### Resource（リソース型）

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `Public/RenderGraphResources.h` | `FRDGTexture`, `FRDGBuffer`, `FRDGView`, `FRDGUAV`, `FRDGSRV` | RDG 管理リソース | [[Reference/ref_rdg_resources]] |
| `Public/RenderGraphResources.inl` | インライン実装 | — | 同上 |
| `Private/RenderGraphResources.cpp` | ライフタイム管理 | 参照カウント・エイリアシング | [[Details/b_rdg_resources]] |
| `Public/RenderGraphTextureSubresource.h` | `FRDGTextureSubresourceRange`, `FRDGTextureSubresourceLayout` | サブリソース範囲 | [[Reference/ref_rdg_texture_subresource]] |

### Pass（パス定義）

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `Public/RenderGraphPass.h` | `FRDGPass`, `TRDGLambdaPass` | パス基底・ラムダパス | [[Reference/ref_rdg_pass]] |
| `Private/RenderGraphPass.cpp` | `FRDGPass::Execute()`, フラグ解釈 | パス実行 | [[Details/c_rdg_pass]] |

### Parameter（パラメータ構造体）

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `Public/RenderGraphParameter.h` | `FRDGParameter`, `FRDGParameterStruct`, `FRDGParameterStructRef` | パラメータラッパー | [[Reference/ref_rdg_parameter]] |
| `Private/RenderGraphParameter.cpp` | アクセサ・イテレータ | — | [[Details/d_rdg_parameters]] |

### Allocator・Blackboard・Utils

| ファイル | 主要クラス | 役割 | 参照 |
|---------|----------|------|------|
| `Public/RenderGraphAllocator.h` | `FRDGAllocator`, `FRDGAllocatorScope` | グラフ内 CPU メモリスタック（TLS） | [[Reference/ref_rdg_allocator]] |
| `Public/RenderGraphBlackboard.h` | `FRDGBlackboard` | パス間データストア（型安全） | [[Reference/ref_rdg_blackboard]] |
| `Public/RenderGraphUtils.h` | `AddCopyTexturePass`, `AddClearUAVPass` ほか多数 | ヘルパー関数群 | [[Reference/ref_rdg_utils]] |
| `Public/RenderGraphDefinitions.h` | `ERDGPassFlags`, `ERDGTextureFlags`, enum 類 | 基本型・フラグ | [[Reference/ref_rdg_definitions]] |

### Event・Validation・Trace

| ファイル | 主要クラス / マクロ | 役割 | 参照 |
|---------|----------------|------|------|
| `Public/RenderGraphEvent.h` | `FRDGEventName`, `RDG_EVENT_NAME`, `RDG_EVENT_SCOPE` | GPU プロファイルスコープ | [[Reference/ref_rdg_event]] |
| `Public/RenderGraphEvent.inl` | インライン | — | 同上 |
| `Public/RenderGraphValidation.h` | `FRDGUserValidation`, `FRDGBarrierValidation` | デバッグ検証 | [[Reference/ref_rdg_validation]] |
| `Public/RenderGraphTrace.h` | `FRDGTrace` | Unreal Insights トレース出力 | [[Reference/ref_rdg_trace]] |
| `Private/RenderGraphPrivate.h` | 内部型 | — | [[Reference/ref_rdg_private]] |

---

## ライフサイクル → ソース対応

```
FRDGBuilder 生成                 SceneRenderBuilder.cpp:872
  │
  ├─ CreateTexture/Buffer        RenderGraphBuilder.h
  │    └─ プール未確保（宣言のみ）
  │
  ├─ AddPass（ラムダ defer）     RenderGraphBuilder.h
  │    └─ TRDGLambdaPass 登録     RenderGraphPass.h
  │
  ├─ Compile                     RenderGraphBuilder.cpp
  │    ├─ パスカリング
  │    ├─ バリア挿入
  │    ├─ RenderPass マージ
  │    └─ AsyncCompute 分割
  │
  └─ Execute                     RenderGraphBuilder.cpp:1755
       ├─ GPU リソース確保（Transient Allocator）
       ├─ パス実行（ラムダ評価 → RHI コマンド）
       └─ QueueTextureExtraction 反映
```

---

## 主要 CVar

| CVar | 定義位置 | 効果 |
|------|--------|------|
| `r.RDG.Debug` | `RenderGraphPrivate.h` | デバッグ情報出力 |
| `r.RDG.Parallel` | 同 | Setup/Compile/Execute 並列化 |
| `r.RDG.ImmediateMode` | 同 | 即時実行（AddPass 時点でラムダ評価） |
| `r.RDG.BreakPoint` | 同 | 特定パスで break |
| `r.RDG.ClobberResources` | 同 | 未初期化リソースをランダム値で埋める |
| `r.RDG.TransientResourceAllocator` | 同 | Transient エイリアシング |

---

## Reference/Details 一覧

| 種別 | ファイル | 対象 |
|------|---------|------|
| Reference | [[Reference/ref_rdg_builder]] | FRDGBuilder API |
| Reference | [[Reference/ref_rdg_resources]] | FRDGTexture/Buffer/View |
| Reference | [[Reference/ref_rdg_pass]] | FRDGPass |
| Reference | [[Reference/ref_rdg_parameter]] | FRDGParameterStruct |
| Reference | [[Reference/ref_rdg_allocator]] | FRDGAllocator |
| Reference | [[Reference/ref_rdg_blackboard]] | FRDGBlackboard |
| Reference | [[Reference/ref_rdg_event]] | RDG_EVENT_* マクロ |
| Reference | [[Reference/ref_rdg_utils]] | ユーティリティ関数 |
| Reference | [[Reference/ref_rdg_definitions]] | enum / 基本型 |
| Reference | [[Reference/ref_rdg_validation]] | デバッグ検証 |
| Reference | [[Reference/ref_rdg_trace]] | Insights トレース |
| Reference | [[Reference/ref_rdg_private]] | 内部型 |
| Reference | [[Reference/ref_rdg_texture_subresource]] | サブリソース範囲 |
| Details | [[Details/a_rdg_builder]] | Execute 内部フロー |
| Details | [[Details/b_rdg_resources]] | リソースライフタイム |
| Details | [[Details/c_rdg_pass]] | パス種別・フラグ |
| Details | [[Details/d_rdg_parameters]] | パラメータ追跡 |
| Details | [[Details/e_rdg_debug_trace]] | デバッグ・トレース |

---

## ue5-dive 起点

- 「バリアがいつ入るか」 → `RenderGraphBuilder.cpp:Compile()` + [[Details/b_rdg_resources]]
- 「AsyncCompute の切り替え」 → `ERDGPassFlags::AsyncCompute` 定義（`RenderGraphDefinitions.h`）
- 「リソースが消える仕組み」 → Transient Resource Allocator（`RenderGraphResources.cpp`）
- 「新しいヘルパーを追加するなら」 → `RenderGraphUtils.h` 既存関数を参考に
