# REF: FMeshPassProcessor

- 対象ファイル: `MeshPassProcessor.h`, `MeshPassProcessor.inl`, `MeshPassProcessor.cpp`
- 関連Details: [[a_mpp_pipeline]]

---

## FMeshPassProcessor — 基底クラス

`MeshPassProcessor.h:2198`

### メンバ変数

| 変数 | 型 | 説明 |
|-----|----|------|
| `MeshPassType` | `EMeshPass::Type` | このプロセッサが担当するパス種別 |
| `Scene` | `const FScene*` | 所属シーン |
| `FeatureLevel` | `ERHIFeatureLevel::Type` | SM5 / SM6 等 |
| `ViewIfDynamicMeshCommand` | `const FSceneView*` | 動的コマンド生成時のビュー（静的時は nullptr）|
| `DrawListContext` | `FMeshPassDrawListContext*` | 書き込み先（静的 or 動的）|

### 主要メソッド

```cpp
// 純粋仮想 — サブクラスが必ず実装
virtual void AddMeshBatch(
    const FMeshBatch& MeshBatch, uint64 BatchElementMask,
    const FPrimitiveSceneProxy* PrimitiveSceneProxy,
    int32 StaticMeshId = -1) = 0;

// コアテンプレート — BuildMeshDrawCommands
template<typename PassShadersType, typename ShaderElementDataType>
void BuildMeshDrawCommands(
    const FMeshBatch& MeshBatch, uint64 BatchElementMask,
    const FPrimitiveSceneProxy* PrimitiveSceneProxy,
    const FMaterialRenderProxy& MaterialRenderProxy,
    const FMaterial& MaterialResource,
    const FMeshPassProcessorRenderState& DrawRenderState,
    const PassShadersType& PassShaders,
    ERasterizerFillMode MeshFillMode, ERasterizerCullMode MeshCullMode,
    FMeshDrawCommandSortKey SortKey,
    EMeshPassFeatures MeshPassFeatures,
    const ShaderElementDataType& ShaderElementData);

// 静的ユーティリティ
static ERasterizerCullMode InverseCullMode(ERasterizerCullMode CullMode);
static ERasterizerFillMode ComputeMeshFillMode(const FMaterial&, const FMeshBatch&, bool bForceWireframe = false);
static ERasterizerCullMode ComputeMeshCullMode(const FMaterial&, const FMeshBatch&, bool bReverseCulling = false);
static FMeshDrawingPolicyOverrideSettings ComputeMeshOverrideSettings(const FMeshBatch&);
```

---

## FMeshPassProcessorRenderState

### メソッド一覧

| メソッド | 説明 |
|---------|------|
| `SetBlendState(FRHIBlendState*)` | ブレンドステートを設定 |
| `SetDepthStencilState(FRHIDepthStencilState*)` | 深度ステンシルステートを設定 |
| `SetStencilRef(uint32)` | ステンシルリファレンス値を設定 |
| `SetViewUniformBuffer(FRHIUniformBuffer*)` | View UB を設定 |
| `SetPassUniformBuffer(FRHIUniformBuffer*)` | パス UB を設定 |
| `SetDepthStencilAccess(FExclusiveDepthStencil::Type)` | 深度ステンシルアクセスモードを設定 |
| `SetNullDepthStencilState()` | 深度ステンシルを無効化 |
| `GetBlendState() const` | ブレンドステート取得 |
| `GetDepthStencilState() const` | 深度ステンシルステート取得 |
| `GetDepthStencilAccess() const` | アクセスモード取得 |
| `ApplyToPSO(FGraphicsPipelineStateInitializer&)` | PSO に全ステートを適用 |

---

## EMeshPassFeatures

| 値 | 説明 |
|----|------|
| `Default` | 通常（全頂点属性）|
| `PositionOnly` | 位置のみ（DepthPass 等で軽量化）|
| `PositionAndNormalOnly` | 位置 + 法線のみ |

---

## FMeshPassDrawListContext（書き込み先インターフェース）

```cpp
class FMeshPassDrawListContext
{
public:
    virtual FMeshDrawCommand& AddCommand(
        FMeshDrawCommand& Initializer, uint32 NumElements) = 0;
    virtual void FinalizeCommand(
        const FMeshBatch& MeshBatch, int32 BatchElementIndex,
        const FMeshDrawCommandPrimitiveIdInfo& IdInfo,
        ERasterizerFillMode MeshFillMode, ERasterizerCullMode MeshCullMode,
        FMeshDrawCommandSortKey SortKey,
        EFVisibleMeshDrawCommandFlags Flags,
        const FGraphicsMinimalPipelineStateInitializer& PipelineState,
        const FMeshProcessorShaders* ShadersForDebugging,
        FMeshDrawCommand& MeshDrawCommand) = 0;
};

// 静的コマンドキャッシュ書き込み先
class FCachedPassMeshDrawListContext : public FMeshPassDrawListContext { ... };

// 動的コマンドフレーム一時バッファ書き込み先
class FDynamicPassMeshDrawListContext : public FMeshPassDrawListContext { ... };
```

---

## パス登録マクロ

```cpp
// パス登録（ShadingPath・MeshPass・MeshPassFlags の紐付け）
REGISTER_MESHPASSPROCESSOR_AND_PSOCOLLECTOR(
    PassName,              // 識別名（ログ等に使用）
    FactoryFunction,       // FMeshPassProcessor* (*Factory)(...)
    EShadingPath,          // Deferred / Mobile / Forward
    EMeshPass::Type,
    EMeshPassFlags)
```

### EMeshPassFlags

| フラグ | 値 | 説明 |
|--------|-----|------|
| `None` | 0 | なし |
| `CachedMeshCommands` | 1 << 0 | 静的コマンドキャッシュを使用 |
| `MainView` | 1 << 1 | メインビューパス |

---

## SubmitMeshDrawCommands

```cpp
// コマンド配列全体を RHI コマンドリストに発行
void SubmitMeshDrawCommands(
    const FMeshCommandOneFrameArray& VisibleMeshDrawCommands,
    const FGraphicsMinimalPipelineStateSet& GraphicsMinimalPipelineStateSet,
    const FMeshDrawCommandSceneArgs& SceneArgs,
    uint32 PrimitiveIdBufferStride,
    bool bDynamicInstancing,
    uint32 InstanceFactor,
    FRHICommandList& RHICmdList);   // MeshPassProcessor.cpp:1604

// 並列分割対応の範囲指定発行
void SubmitMeshDrawCommandsRange(
    const FMeshCommandOneFrameArray& VisibleMeshDrawCommands,
    const FGraphicsMinimalPipelineStateSet& GraphicsMinimalPipelineStateSet,
    const FMeshDrawCommandSceneArgs& SceneArgs,
    uint32 PrimitiveIdBufferStride,
    bool bDynamicInstancing,
    int32 StartIndex, int32 NumMeshDrawCommands,
    uint32 InstanceFactor,
    FRHICommandList& RHICmdList);   // MeshPassProcessor.cpp:1616
```

---

> [!note]- BuildMeshDrawCommands はヘッダ定義のテンプレート
> `BuildMeshDrawCommands` は `MeshPassProcessor.h:2247` に定義されたテンプレート関数。  
> `PassShadersType` によってコンパイル時にシェーダー型が決まるため、`.inl` ファイルで実装が展開される。  
> テンプレートのため `MeshPassProcessor.cpp` には存在しない点に注意。

> [!note]- ViewIfDynamicMeshCommand の nullptr チェック
> 静的コマンドキャッシュ（シーン追加時）では `ViewIfDynamicMeshCommand == nullptr`。  
> タービランス等のカメラ依存パラメータは動的コマンド生成時にのみ使える。  
> シェーダーバインディングでカメラ依存値が必要なら必ず nullptr チェックを行う。

> [!note]- FMeshPassDrawListContext の使い分け
> `DrawListContext` は `FMeshPassProcessor` のコンストラクタ引数で渡される。  
> - `FCachedPassMeshDrawListContextImmediate` — シーン追加時の静的キャッシュ用  
> - `FCachedPassMeshDrawListContextDeferred` — 遅延静的キャッシュ用（マルチスレッドビルド）  
> - `FDynamicPassMeshDrawListContext` — 毎フレーム動的コマンド用  
> 実装を変えることで同じ `AddMeshBatch()` 呼び出しが異なる格納先に書き込む。
