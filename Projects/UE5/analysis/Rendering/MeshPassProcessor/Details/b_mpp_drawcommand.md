# B: FMeshDrawCommand

- 対象: `MeshDrawCommands.h/.cpp`
- 上位: [[11_mpp_overview]]
- Reference: [[ref_mpp_drawcommand]]

---

## FMeshDrawCommand とは

RHI に直接渡せる最小描画単位。  
`BuildMeshDrawCommands()` によって生成され、静的キャッシュまたはフレーム一時バッファに格納される。  
`SubmitMeshDrawCommands()` が RHI コマンドリストに変換して実際の DrawCall を発行する。

---

## データ構造

```cpp
class FMeshDrawCommand
{
public:
    // ─── シェーダーバインディング ───
    FMeshDrawShaderBindings ShaderBindings;   // VS/PS/CS 等全シェーダーのバインド情報

    // ─── 頂点ストリーム ───
    FVertexInputStreamArray VertexStreams;     // VB・ストライド・オフセット

    // ─── インデックスバッファ ───
    FRHIBuffer* IndexBuffer;                  // nullptr = 非インデックス描画

    // ─── パイプラインステートオブジェクト ───
    FGraphicsMinimalPipelineStateId CachedPipelineId; // PSO キャッシュへのインデックス

    // ─── 描画引数 ───
    uint32 FirstIndex;       // IB の開始インデックス
    uint32 NumPrimitives;    // プリミティブ数（0 = DrawIndirect）
    uint32 NumInstances;     // インスタンス数

    union {
        // 通常描画
        struct {
            uint32 BaseVertexIndex;
            uint32 NumVertices;
        } VertexParams;

        // 間接描画（NumPrimitives == 0 の場合）
        struct {
            FRHIBuffer* Buffer;
            uint32 Offset;
        } IndirectArgs;
    };

    // ─── 追加情報 ───
    int8  PrimitiveIdStreamIndex;   // GPUScene のプリミティブ ID ストリームのインデックス
    uint8 StencilRef;
    EPrimitiveType PrimitiveType : PT_NumBits;
};
```

---

## FMeshDrawCommandSortKey — ソートキー

DrawCall をバッチングしやすい順にソートするためのキー。  
`BuildMeshDrawCommands()` の引数として渡す。

```cpp
class FMeshDrawCommandSortKey
{
public:
    union {
        uint64 PackedData;

        // BasePass ソート（シェーダーの変更を最小化）
        struct {
            uint64 VertexShaderHash  : 16; // VS ハッシュ上位ビット
            uint64 PixelShaderHash   : 32; // PS ハッシュ
            uint64 Masked            : 1;  // Masked マテリアルを後ろに
            uint64 PipelineId        : 15; // PSO ID
        } BasePass;

        // Translucent ソート（後ろ→前の Z ソート）
        struct {
            uint64 MeshIdInPrimitive : 16;
            uint64 Distance          : 32; // カメラ距離（逆順）
            uint64 Priority          : 16; // マテリアル優先度
        } Translucent;
    };

    static FMeshDrawCommandSortKey GetDefaultTranslucent(
        const FPrimitiveSceneProxy*, const FMeshBatch&, const FMaterial&);
};
```

---

## SubmitMeshDrawCommands — DrawCall 発行

```cpp
// VisibleMeshDrawCommands 配列全体を RHI に送る
void SubmitMeshDrawCommands(
    const FMeshCommandOneFrameArray& VisibleMeshDrawCommands,
    const FGraphicsMinimalPipelineStateSet& GraphicsMinimalPipelineStateSet,
    FRHIBuffer* SceneArgs,
    const void* ViewIdentity,
    bool bDynamicInstancing,
    uint32 InstanceFactor,
    FRHICommandList& RHICmdList);

// 単一コマンドの送信（内部で使われる）
bool FMeshDrawCommand::SubmitDrawBegin(
    const FMeshDrawCommand& MeshDrawCommand,
    const FGraphicsMinimalPipelineStateSet& GraphicsMinimalPipelineStateSet,
    FRHIBuffer* SceneArgs, uint32 SceneInstanceOffset, uint32 NumInstances,
    FRHICommandList& RHICmdList);

void FMeshDrawCommand::SubmitDrawEnd(
    const FMeshDrawCommand& MeshDrawCommand, uint32 NumInstances,
    FRHICommandList& RHICmdList);

// 上記2つをまとめたショートカット
bool FMeshDrawCommand::SubmitDraw(
    const FMeshDrawCommand& MeshDrawCommand,
    const FGraphicsMinimalPipelineStateSet& PipelineStateSet,
    FRHIBuffer* SceneArgs, uint32 SceneInstanceOffset, uint32 NumInstances,
    FRHICommandList& RHICmdList);
```

---

## ダイナミックインスタンシング

同一 PSO・バインディングを持つコマンドを自動的にインスタンス化してまとめる最適化。

```cpp
// 2つのコマンドがインスタンシング可能か判定
bool FMeshDrawCommand::MatchesForDynamicInstancing(const FMeshDrawCommand& Rhs) const;

// インスタンシング用ハッシュキー
uint32 FMeshDrawCommand::GetDynamicInstancingHash() const;
```

ダイナミックインスタンシングの条件:
- 同じ PSO（`CachedPipelineId`）
- 同じ頂点ストリーム・インデックスバッファ
- バインディングの違いは GPUScene の InstanceData のみ

---

## コマンドの生成から発行までの流れ

```
BuildMeshDrawCommands() 内:
  1. FGraphicsPipelineStateInitializer を構築
     ← BoundShaderState (VS/PS), BlendState, DepthStencilState, RasterizerState
  2. PipelineStateCache::FindOrCreateGraphicsPipelineState() で PSO 取得
  3. FMeshDrawCommand::InitializeShaderBindings()
  4. FMeshDrawSingleShaderBindings で各シェーダーのリソースをバインド
  5. FMeshDrawCommand::Finalize()
     ← CachedPipelineId を決定
  6. DrawListContext->FinalizeCommand() で格納先に追加

SubmitMeshDrawCommands() 内:
  1. ソートキーでコマンド配列をソート
  2. ダイナミックインスタンシング判定・グループ化
  3. FMeshDrawCommand::SubmitDraw() でループ発行
     a. SetGraphicsPipelineState() ← PSO セット
     b. シェーダーバインディングを RHI コマンドに変換
     c. RHICmdList.DrawIndexedPrimitive() / DrawPrimitive()
```

---

## デバッグ情報

`MESH_DRAW_COMMAND_DEBUG_DATA` マクロ（非 Shipping/Test ビルド）が有効な場合、  
コマンドにデバッグ文字列が付加される。

```cpp
#define MESH_DRAW_COMMAND_DEBUG_DATA \
    ((!UE_BUILD_SHIPPING && !UE_BUILD_TEST) || VALIDATE_MESH_COMMAND_BINDINGS \
     || WANTS_DRAW_MESH_EVENTS || WITH_DEBUG_VIEW_MODES)
```

---

## コード実行フロー

### エントリポイント

```
─── 静的コマンド生成（シーン追加時 1 回のみ）───────────────────────
FPrimitiveSceneInfo::AddStaticMeshes()          PrimitiveSceneInfo.cpp:1537
  └─ CacheMeshDrawCommands()                    PrimitiveSceneInfo.cpp:583
       └─ FPassProcessorManager::CreateMeshPassProcessor()
            └─ AddMeshBatch() → BuildMeshDrawCommands()
                 └─ FCachedPassMeshDrawListContextImmediate::FinalizeCommand()
                      MeshPassProcessor.cpp:2032
                      → FCachedPassMeshDrawList に永続保存

─── 動的コマンド生成（毎フレーム）──────────────────────────────────
GatherDynamicMeshElements()
  └─ FMeshPassProcessor::AddMeshBatch()
       └─ BuildMeshDrawCommands()
            └─ FDynamicPassMeshDrawListContext::FinalizeCommand()
                 → FMeshCommandOneFrameArray に追加

─── DrawCall 発行（毎フレーム）──────────────────────────────────────
SubmitMeshDrawCommands()              MeshPassProcessor.cpp:1604
  └─ SubmitMeshDrawCommandsRange()   MeshPassProcessor.cpp:1616
       └─ FMeshDrawCommand::SubmitDraw()
            ├─ SubmitDrawBegin()   PSO セット・バインディング適用
            └─ SubmitDrawEnd()     DrawIndexedPrimitive / DrawPrimitive
```

### フロー詳細

1. **静的コマンドキャッシュ生成** `PrimitiveSceneInfo.cpp:583`  
   `FPrimitiveSceneInfo::CacheMeshDrawCommands()` が全 EMeshPass に対して `FMeshPassProcessor` を生成し、静的メッシュの `FMeshBatch` を処理。結果は `FScene` の `FCachedPassMeshDrawList` に保存され、プリミティブが変化しない限り再利用される。

2. **可視コマンドのコピー** （フレーム初期化フェーズ）  
   `FViewInfo::VisibleMeshDrawCommands[EMeshPass]` に、`PrimitiveVisibilityMap` で可視になった静的コマンドがコピーされる。動的コマンドはこのフレームで同配列に直接追加される。

3. **ソート**  
   `FMeshDrawCommandSortKey::PackedData` の昇順でコマンド配列をソート。BasePass は PSO 変更最小化（シェーダーハッシュ順）、半透明は後ろ→前の Z ソート。

4. **ダイナミックインスタンシング判定**  
   `FMeshDrawCommand::MatchesForDynamicInstancing()` `MeshPassProcessor.cpp:1018` で同一 PSO・頂点ストリーム・バインディングのコマンドをグループ化し、インスタンスとしてまとめる。

5. **SubmitMeshDrawCommandsRange()** `MeshPassProcessor.cpp:1616`  
   コマンドをループし、各コマンドに対して `FMeshDrawCommand::SubmitDraw()` を呼ぶ。`FMeshDrawCommandStateCache` でコマンド間の PSO・バインディングの差分のみを適用する。

6. **FMeshDrawCommand::SubmitDraw()** `MeshPassProcessor.h:1416`  
   - `SubmitDrawBegin()`: `SetGraphicsPipelineState()` + `ShaderBindings.SetOnCommandList()` + プリミティブ ID バッファ設定
   - `SubmitDrawEnd()`: `DrawIndexedPrimitive()` または `DrawPrimitive()`

### 関与クラス・関数一覧

| クラス / 関数 | ファイル:行 | 説明 |
|--------------|------------|------|
| `FPrimitiveSceneInfo::CacheMeshDrawCommands()` | `PrimitiveSceneInfo.cpp:583` | 静的コマンドキャッシュの生成 |
| `FPrimitiveSceneInfo::AddStaticMeshes()` | `PrimitiveSceneInfo.cpp:1537` | シーン追加時の静的メッシュ登録エントリ |
| `FCachedPassMeshDrawListContextImmediate::FinalizeCommand()` | `MeshPassProcessor.cpp:2032` | 静的キャッシュへの書き込み |
| `SubmitMeshDrawCommands()` | `MeshPassProcessor.cpp:1604` | コマンド配列全体の発行 |
| `SubmitMeshDrawCommandsRange()` | `MeshPassProcessor.cpp:1616` | 範囲指定の発行（並列分割用）|
| `FMeshDrawCommand::SubmitDraw()` | `MeshPassProcessor.h:1416` | 1 コマンドの PSO セット + DrawCall |
| `FMeshDrawCommand::SubmitDrawBegin()` | `MeshPassProcessor.h:1387` | PSO・バインディング適用 |
| `FMeshDrawCommand::SubmitDrawEnd()` | `MeshPassProcessor.h:1397` | DrawIndexedPrimitive 発行 |
| `FMeshDrawCommand::MatchesForDynamicInstancing()` | `MeshPassProcessor.cpp:1018` | 動的インスタンシング判定 |
| `CalculateMeshStaticSortKey()` | `MeshPassProcessor.cpp:1772` | VS/PS ハッシュからソートキー生成 |
