# A: MeshPass パイプライン全体

- 対象: `MeshPassProcessor.h/.inl/.cpp`
- 上位: [[11_mpp_overview]]
- Reference: [[ref_mpp_processor]]

---

## パイプライン概要

```
FMeshBatch（メッシュの描画要求）
  │
  ▼
FMeshPassProcessor::AddMeshBatch()   ← サブクラスがオーバーライド
  │
  ▼
BuildMeshDrawCommands()              ← 基底クラスが提供
  ├─ マテリアル解決（FMaterialRenderProxy → FMaterial）
  ├─ シェーダー取得（VertexShader / PixelShader / etc.）
  ├─ PSO 構築（FGraphicsPipelineStateInitializer）
  ├─ バインディング設定（FMeshDrawShaderBindings）
  └─ FMeshDrawCommand 生成 → DrawListContext に追加
```

---

## FMeshPassProcessor — 基底クラス

```cpp
class FMeshPassProcessor : public IPSOCollector
{
public:
    // 必須: サブクラスで実装
    virtual void AddMeshBatch(
        const FMeshBatch& MeshBatch,
        uint64 BatchElementMask,
        const FPrimitiveSceneProxy* PrimitiveSceneProxy,
        int32 StaticMeshId = -1) = 0;

    // コンテキスト情報
    EMeshPass::Type       MeshPassType;
    const FScene*         Scene;
    ERHIFeatureLevel::Type FeatureLevel;
    const FSceneView*     ViewIfDynamicMeshCommand;  // 動的コマンドの場合のみ有効
    FMeshPassDrawListContext* DrawListContext;

    // ユーティリティ
    static ERasterizerFillMode ComputeMeshFillMode(const FMaterial&, ...);
    static ERasterizerCullMode ComputeMeshCullMode(const FMaterial&, ...);
    static ERasterizerCullMode InverseCullMode(ERasterizerCullMode);
    static FMeshDrawingPolicyOverrideSettings ComputeMeshOverrideSettings(const FMeshBatch&);
};
```

---

## BuildMeshDrawCommands — コア処理

サブクラスが `AddMeshBatch()` 内で呼び出す主要メソッド。  
PSO・シェーダーバインディング・描画引数をまとめて `FMeshDrawCommand` に変換する。

```cpp
template<typename PassShadersType, typename ShaderElementDataType>
void FMeshPassProcessor::BuildMeshDrawCommands(
    const FMeshBatch& MeshBatch,
    uint64 BatchElementMask,
    const FPrimitiveSceneProxy* PrimitiveSceneProxy,
    const FMaterialRenderProxy& MaterialRenderProxy,
    const FMaterial& MaterialResource,
    const FMeshPassProcessorRenderState& DrawRenderState,
    const PassShadersType& PassShaders,         // VS/PS/GS/MS 等をまとめた構造体
    ERasterizerFillMode MeshFillMode,
    ERasterizerCullMode MeshCullMode,
    FMeshDrawCommandSortKey SortKey,
    EMeshPassFeatures MeshPassFeatures,
    const ShaderElementDataType& ShaderElementData);
```

### 典型的なサブクラス実装例（BasePass 風）

```cpp
void FMyMeshPassProcessor::AddMeshBatch(
    const FMeshBatch& MeshBatch, uint64 BatchElementMask,
    const FPrimitiveSceneProxy* Proxy, int32 StaticMeshId)
{
    // マテリアル取得
    const FMaterialRenderProxy* MatProxy = MeshBatch.MaterialRenderProxy;
    const FMaterial* Mat = MatProxy->GetMaterialNoFallback(FeatureLevel);
    if (!Mat) return;

    // シェーダー取得
    TMeshProcessorShaders<FMyVS, FMyPS> Shaders;
    Shaders.VertexShader = Mat->GetShader<FMyVS>(MeshBatch.VertexFactory->GetType());
    Shaders.PixelShader  = Mat->GetShader<FMyPS>();

    // レンダーステート
    FMeshPassProcessorRenderState DrawRenderState;
    DrawRenderState.SetDepthStencilState(TStaticDepthStencilState<true, CF_DepthNearOrEqual>::GetRHI());

    // ソートキー（不透明: シェーダーハッシュ、半透明: 深度）
    FMeshDrawCommandSortKey SortKey = CreateSortKey();

    // コマンド生成
    BuildMeshDrawCommands(MeshBatch, BatchElementMask, Proxy,
        *MatProxy, *Mat, DrawRenderState, Shaders,
        FM_Solid, CM_CCW, SortKey,
        EMeshPassFeatures::Default, /* ShaderElementData */ FMeshMaterialShaderElementData());
}
```

---

## FMeshPassProcessorRenderState — レンダーステート

```cpp
struct FMeshPassProcessorRenderState
{
    // ブレンドステート
    void SetBlendState(FRHIBlendState* State);
    FRHIBlendState* GetBlendState() const;

    // 深度ステンシルステート
    void SetDepthStencilState(FRHIDepthStencilState* State);
    void SetStencilRef(uint32 Ref);
    FRHIDepthStencilState* GetDepthStencilState() const;
    FExclusiveDepthStencil::Type GetDepthStencilAccess() const;

    // Uniform Buffer
    void SetViewUniformBuffer(FRHIUniformBuffer* UB);
    void SetPassUniformBuffer(FRHIUniformBuffer* UB);

    // PSO に適用
    void ApplyToPSO(FGraphicsPipelineStateInitializer& PSO) const;
};
```

---

## EMeshPassFeatures — メッシュパス特性

```cpp
enum class EMeshPassFeatures {
    Default             = 0,
    PositionOnly        = 1 << 0,  // 位置のみ（DepthPass 等）
    PositionAndNormalOnly = 1 << 1, // 位置 + 法線のみ
};
```

`PositionOnly` を使うと頂点シェーダーが軽量になる（Shadow / Depth 専用）。

---

## パス登録マクロ

新しいパスを登録する際のマクロ:

```cpp
REGISTER_MESHPASSPROCESSOR_AND_PSOCOLLECTOR(
    FMyPassName,          // 識別名
    &CreateMyProcessor,   // FMeshPassProcessor* (*)(args...) ファクトリ関数
    EShadingPath::Deferred,
    EMeshPass::MyPass,
    EMeshPassFlags::CachedMeshCommands);
```

---

## 静的コマンドキャッシュの仕組み

```
[シーン追加時: 静的メッシュのコマンドキャッシュ]

FPrimitiveSceneInfo::CacheMeshDrawCommands()
  └─ 各 EMeshPass に対して FMeshPassProcessor を生成
       └─ AddMeshBatch() → BuildMeshDrawCommands()
            └─ FCachedMeshDrawCommands に保存

[毎フレーム: キャッシュされたコマンドを再利用]

FVisibleMeshDrawCommandsArray に静的コマンドをコピー
  └─ SubmitMeshDrawCommands() で DrawCall 発行
       ← シェーダーバインディングの再設定コストなし
```

---

## DrawListContext の種類

```cpp
// 静的コマンドのキャッシュ書き込み先
class FCachedPassMeshDrawListContext : public FMeshPassDrawListContext { ... };

// 動的コマンドのフレーム一時リスト書き込み先
class FDynamicPassMeshDrawListContext : public FMeshPassDrawListContext { ... };
```

---

## コード実行フロー

### エントリポイント

```
[サブクラス::AddMeshBatch()]         MeshPassProcessor.h:2224
  │  マテリアル・シェーダー解決
  │  PSO パラメータ確定
  │
  └─ BuildMeshDrawCommands<PS,SD>()  MeshPassProcessor.h:2247
       │
       ├─ InitializeShaderBindings()   バインディング領域確保
       ├─ GetShaderBindings()×シェーダー数
       ├─ SetDrawParametersAndFinalize()  PSO ID・描画引数確定
       │
       └─ DrawListContext->FinalizeCommand()
            │
            ├─ [静的] FCachedPassMeshDrawListContextImmediate::FinalizeCommand()
            │         MeshPassProcessor.cpp:2032
            │         → FCachedPassMeshDrawList::MeshDrawCommands に保存
            │
            └─ [動的] FDynamicPassMeshDrawListContext::FinalizeCommand()
                      MeshPassProcessor.h:1820
                      → FMeshCommandOneFrameArray に追加
```

### フロー詳細

1. **AddMeshBatch()** `MeshPassProcessor.h:2224`  
   サブクラスが実装する純粋仮想関数。`FMeshBatch` からマテリアル・シェーダーを解決し、`BuildMeshDrawCommands()` を呼ぶ。

2. **BuildMeshDrawCommands()** `MeshPassProcessor.h:2247`  
   テンプレート関数。`PassShadersType` にシェーダーセット、`ShaderElementDataType` に LOD フェード等の要素データを受け取る。内部で以下を順に実行する:
   - `FGraphicsMinimalPipelineStateInitializer` 構築（BoundShaderState + BlendState + DepthStencilState + RasterizerState）
   - `FMeshDrawCommand::InitializeShaderBindings(Shaders)` でバインディング領域を確保
   - シェーダーごとに `GetSingleShaderBindings()` → `GetShaderBindings()` でリソースをバインド
   - `SetDrawParametersAndFinalize()` で `CachedPipelineId` と描画引数を確定

3. **DrawListContext->FinalizeCommand()** `MeshPassProcessor.h:1677`  
   `DrawListContext` の実装に応じて格納先が変わる:
   - **静的パス**: `FCachedPassMeshDrawListContextImmediate::FinalizeCommand` `MeshPassProcessor.cpp:2032` → `FScene` の `FCachedPassMeshDrawList` に永続保存
   - **動的パス**: `FDynamicPassMeshDrawListContext::FinalizeCommand` `MeshPassProcessor.h:1820` → `FMeshCommandOneFrameArray`（フレーム一時）に追加

### 関与クラス・関数一覧

| クラス / 関数 | ファイル:行 | 説明 |
|--------------|------------|------|
| `FMeshPassProcessor::AddMeshBatch()` | `MeshPassProcessor.h:2224` | 純粋仮想。サブクラスがオーバーライド |
| `FMeshPassProcessor::BuildMeshDrawCommands()` | `MeshPassProcessor.h:2247` | PSO・バインディング・コマンド生成コア |
| `FMeshDrawCommand::InitializeShaderBindings()` | `MeshPassProcessor.h` | シェーダーバインディング領域の確保 |
| `FMeshDrawCommand::SetDrawParametersAndFinalize()` | `MeshPassProcessor.h` | 描画引数と PSO ID を確定 |
| `FMeshPassDrawListContext::FinalizeCommand()` | `MeshPassProcessor.h:1677` | 格納先への書き込み（仮想）|
| `FCachedPassMeshDrawListContextImmediate::FinalizeCommand()` | `MeshPassProcessor.cpp:2032` | 静的キャッシュへの書き込み |
| `FDynamicPassMeshDrawListContext::FinalizeCommand()` | `MeshPassProcessor.h:1820` | 動的フレームリストへの追加 |
| `FPassProcessorManager::CreateMeshPassProcessor()` | `MeshPassProcessor.h:2353` | EMeshPass からファクトリ経由でプロセッサを生成 |
