# リファレンス：LumenSceneGPUDrivenUpdate.h / LumenSceneGPUDrivenUpdate.cpp

- グループ: a - Surface Cache
- 上位: [[a_lumen_surface_cache]]
- 関連: [[ref_lumen_scene_data]] | [[ref_lumen_scene]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneGPUDrivenUpdate.h/cpp`

---

## 概要

GPU Driven な Lumen シーン更新システム。  
プリミティブの**可視性判定・MeshCards の追加/削除**を GPU Compute で行い、  
結果を CPU にリードバックして次フレームの Surface Cache 更新に反映する。

従来の CPU 側ループと比べて、大量プリミティブ環境でのパフォーマンスを改善する。

---

## 従来方式との比較

| 比較項目 | CPU Driven（旧）| GPU Driven（新）|
|---------|----------------|----------------|
| 可視性判定 | CPU で全プリミティブをループ | GPU コンピュートで並列処理 |
| 更新レイテンシ | 即時 | 数フレーム遅延（リードバック待ち）|
| 大量プリミティブ | CPU ボトルネックになりやすい | GPU で高速処理 |
| `r.LumenScene.ParallelUpdate` | マルチスレッド CPU 処理 | GPU Compute と組み合わせ |

---

## FLumenSceneReadback

> **概要**: GPU から CPU へのリードバックバッファ管理クラス。`FLumenSurfaceCacheFeedback` と同様のリングバッファ方式（最大 4 フライト中）。AddOps / RemoveOps の 2 種類のバッファを管理する。

```cpp
class FLumenSceneReadback : public FRenderResource {
public:
    struct FAddOp    { ... };
    struct FRemoveOp { ... };
    struct FBuffersRHI { ... };
    struct FBuffersRDG { ... };

    FBuffersRDG GetWriteBuffers(FRDGBuilder& GraphBuilder);
    void SubmitWriteBuffers(FRDGBuilder& GraphBuilder, const FBuffersRDG& BuffersRDG);
    const FBuffersRHI* GetLatestReadbackBuffers();

    static int32 GetMaxAddOps();
    static int32 GetMaxRemoveOps();
    static int32 GetAddOpsBufferSizeInBytes();
    static int32 GetRemoveOpsBufferSizeInBytes();
};
```

---

## FLumenSceneReadback::FAddOp

GPU からの「MeshCards 追加リクエスト」。

```cpp
struct FAddOp {
    uint32 PrimitiveGroupIndex;  // 追加対象の PrimitiveGroup インデックス
    float  DistanceSq;           // カメラからの距離の 2 乗（優先度付きソート用）
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `PrimitiveGroupIndex` | `uint32` | 追加する `FLumenPrimitiveGroup` のシーンインデックス |
| `DistanceSq` | `float` | カメラ原点からプリミティブ AABB までの距離の 2 乗（近い順に優先処理）|

### 使用箇所
- [[ref_lumen_scene_gpu_driven_update]] `FLumenSceneReadback::GetLatestReadbackBuffers()` でリードバック後、CPU が読み取って `FLumenSceneData::AddMeshCards()` を呼ぶ

---

## FLumenSceneReadback::FRemoveOp

GPU からの「MeshCards 削除リクエスト」。

```cpp
struct FRemoveOp {
    uint32 PrimitiveGroupIndex;  // 削除対象の PrimitiveGroup インデックス
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `PrimitiveGroupIndex` | `uint32` | 削除する `FLumenPrimitiveGroup` のシーンインデックス |

---

## FLumenSceneReadback::FBuffersRHI

RHI レイヤーでのリードバックバッファへの参照。

```cpp
struct FBuffersRHI {
    FRHIGPUBufferReadback* AddOps;    // AddOp リードバックバッファ
    FRHIGPUBufferReadback* RemoveOps; // RemoveOp リードバックバッファ
};
```

### 使用箇所
- [[ref_lumen_scene_gpu_driven_update]] `GetLatestReadbackBuffers()` の戻り値

---

## FLumenSceneReadback::FBuffersRDG

RDG パス内での書き込み用バッファの参照。

```cpp
struct FBuffersRDG {
    FRDGBufferRef AddOps;    // GPU コンピュートが AddOp を書き込む RDG バッファ
    FRDGBufferRef RemoveOps; // GPU コンピュートが RemoveOp を書き込む RDG バッファ
};
```

---

## FLumenSceneReadback::GetWriteBuffers

```cpp
FBuffersRDG GetWriteBuffers(FRDGBuilder& GraphBuilder);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー（バッファ生成または既存バッファを登録）|

### 戻り値
`FBuffersRDG` — GPU が書き込む用の AddOps・RemoveOps RDG バッファ参照

### 使用箇所
- [[ref_lumen_scene_gpu_driven_update]] `LumenScene::GPUDrivenUpdate()` — GPU コンピュートパスに渡す書き込みターゲットを取得

### 内部動作
```cpp
FBuffersRDG FLumenSceneReadback::GetWriteBuffers(FRDGBuilder& GraphBuilder)
{
    // リングバッファの書き込み先インデックス
    int32 WriteIndex = ReadbackBuffersWriteIndex;

    FBuffersRDG Result;
    // 既存の RHI バッファを RDG に登録して返す
    Result.AddOps    = GraphBuilder.RegisterExternalBuffer(AddOpsBuffers[WriteIndex]);
    Result.RemoveOps = GraphBuilder.RegisterExternalBuffer(RemoveOpsBuffers[WriteIndex]);
    return Result;
}
```

---

## FLumenSceneReadback::SubmitWriteBuffers

```cpp
void SubmitWriteBuffers(FRDGBuilder& GraphBuilder, const FBuffersRDG& BuffersRDG);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー（エンキューコピーパス追加用）|
| `BuffersRDG` | `const FBuffersRDG&` | GPU が書き込んだ RDG バッファ |

### 使用箇所
- [[ref_lumen_scene_gpu_driven_update]] `LumenScene::GPUDrivenUpdate()` — GPU コンピュート完了後にリードバックをキューに投入

### 内部処理フロー

1. **GPU → CPU リードバックのキューイング**
   ```cpp
   AddReadback = ReadbackBuffers[WriteIndex].AddOps;
   AddReadback->EnqueueCopy(GraphBuilder, BuffersRDG.AddOps,    GetAddOpsBufferSizeInBytes());
   RemoveReadback->EnqueueCopy(GraphBuilder, BuffersRDG.RemoveOps, GetRemoveOpsBufferSizeInBytes());
   ```

2. **リングバッファインデックスの更新**
   ```cpp
   ReadbackBuffersWriteIndex = (ReadbackBuffersWriteIndex + 1) % MaxReadbackBuffers;
   ReadbackBuffersNumPending++;
   ```

---

## FLumenSceneReadback::GetLatestReadbackBuffers

```cpp
const FBuffersRHI* GetLatestReadbackBuffers();
```

### 戻り値
`const FBuffersRHI*` — CPU で読み取り可能な最新バッファ（null = まだ準備できていない）

### 使用箇所
- [[ref_lumen_scene]] `UpdateLumenScenePrimitives()` 内で AddOps / RemoveOps を読み取って CPU 側シーンを更新

### 内部処理フロー

```cpp
const FBuffersRHI* FLumenSceneReadback::GetLatestReadbackBuffers()
{
    if (ReadbackBuffersNumPending == 0) return nullptr;

    int32 ReadIndex = (ReadbackBuffersWriteIndex - ReadbackBuffersNumPending
                       + MaxReadbackBuffers) % MaxReadbackBuffers;
    FBuffersRHI& Oldest = ReadbackBuffers[ReadIndex];

    if (!Oldest.AddOps->IsReady() || !Oldest.RemoveOps->IsReady()) {
        return nullptr; // GPU まだ処理中
    }

    ReadbackBuffersNumPending--;
    return &Oldest;
}
```

---

> [!note]- GetMaxAddOps / GetMaxRemoveOps / GetXxxBufferSizeInBytes — バッファサイズ定数
> 
> ```cpp
> static int32 GetMaxAddOps();             // = 1024
> static int32 GetMaxRemoveOps();          // = 1024
> static int32 GetAddOpsBufferSizeInBytes();    // = sizeof(FAddOp)    * GetMaxAddOps()
> static int32 GetRemoveOpsBufferSizeInBytes(); // = sizeof(FRemoveOp) * GetMaxRemoveOps()
> ```
> 
> 1 フレームあたりの AddOps / RemoveOps の上限はそれぞれ 1024 エントリ。

---

## プライベートメンバ

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `MaxReadbackBuffers` | `const int32` | フライト中バッファ数の上限（= 4）|
| `ReadbackBuffersWriteIndex` | `int32` | 現在の書き込み先インデックス（0〜3）|
| `ReadbackBuffersNumPending` | `int32` | 読み取り待ちバッファ数（0〜4）|
| `ReadbackBuffers` | `TArray<FBuffersRHI>` | RHI リードバックバッファリスト（4 セット）|
| `AddOpsBuffers` | `TRefCountPtr<FRDGPooledBuffer>[4]` | AddOps GPU バッファ（永続、RDG に登録して使う）|
| `RemoveOpsBuffers` | `TRefCountPtr<FRDGPooledBuffer>[4]` | RemoveOps GPU バッファ（同上）|

---

## namespace LumenScene — GPUDrivenUpdate

```cpp
namespace LumenScene {
    void GPUDrivenUpdate(
        FRDGBuilder& GraphBuilder,
        const FScene* Scene,
        TArray<FViewInfo>& Views,
        const FLumenSceneFrameTemporaries& FrameTemporaries);
}
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー（コンピュートパスとリードバックパスを追加）|
| `Scene` | `const FScene*` | シーン（`FLumenSceneData` と `PrimitiveGroups` を参照）|
| `Views` | `TArray<FViewInfo>&` | ビュー一覧（フラスタム・カメラ位置の取得用）|
| `FrameTemporaries` | `const FLumenSceneFrameTemporaries&` | `PrimitiveGroupBufferSRV` などの GPU バッファ参照 |

### 使用箇所
- [[ref_lumen_scene]] `FDeferredShadingRenderer::UpdateLumenScene()` — `UpdateLumenScenePrimitives()` の後、`UpdateSurfaceCacheAllocationState()` の前に呼ばれる

### 内部処理フロー

1. **GPU Compute でプリミティブのフラスタムカリング**
   ```cpp
   // GPU パラメータ設定
   PassParams->PrimitiveGroupBuffer = FrameTemporaries.PrimitiveGroupBufferSRV;
   PassParams->ViewFrustumPlanes    = GetFrustumPlanes(Views);
   PassParams->RWAddOps    = GraphBuilder.CreateUAV(WriteBuffers.AddOps);
   PassParams->RWRemoveOps = GraphBuilder.CreateUAV(WriteBuffers.RemoveOps);

   // GPU コンピュートシェーダー実行
   // 各 PrimitiveGroup の ワールド AABB とフラスタムを比較
   // 可視→不可視になったグループ → RemoveOps に書き込み
   // 不可視→可視になったグループ → AddOps に書き込み（距離も書き込み）
   GraphBuilder.AddPass(RDG_EVENT_NAME("GPUDrivenUpdate"), PassParams,
       ERDGPassFlags::Compute, [](FRHIComputeCommandList& RHICmdList) {
           // FLumenGPUDrivenUpdateCS::DispatchGroupsOf64
       });
   ```

2. **リードバックをキューに投入**
   ```cpp
   LumenData.SceneReadback.SubmitWriteBuffers(GraphBuilder, WriteBuffers);
   ```

3. **CPU側: 準備ができたリードバックを読み取って反映（次のフレームで）**
   ```cpp
   // UpdateLumenScenePrimitives() 内で
   const FLumenSceneReadback::FBuffersRHI* ReadbackBuffers =
       LumenData.SceneReadback.GetLatestReadbackBuffers();
   if (ReadbackBuffers) {
       // AddOps を処理: カメラ距離近い順にソートして AddMeshCards()
       // RemoveOps を処理: RemoveMeshCards()
   }
   ```

---

## namespace LumenScene — 補助関数群

```cpp
namespace LumenScene {
    float GetCardMaxDistance(const FViewInfo& View);
    float GetCardTexelDensity();
    float GetFarFieldCardTexelDensity();
    float GetFarFieldCardMaxDistance();
    int32 GetCardMinResolution(bool bOrthographicCamera);
}
```

### 各関数の説明

| 関数 | 戻り値 | 説明 |
|------|--------|------|
| `GetCardMaxDistance(View)` | `float` | カメラ FOV と `r.LumenScene.CardCaptureResolution` から Card のキャプチャ最大距離を返す |
| `GetCardTexelDensity()` | `float` | テクセル/cm の密度（`r.LumenScene.CardTexelDensity` 由来）|
| `GetFarFieldCardTexelDensity()` | `float` | 遠距離フィールド Card のテクセル密度（通常より低い）|
| `GetFarFieldCardMaxDistance()` | `float` | `r.LumenScene.FarField.MaxTraceDistance` の値 |
| `GetCardMinResolution(bOrtho)` | `int32` | Card の最小解像度（正投影カメラは透視より大きい値）|

### 使用箇所
- [[ref_lumen_surface_cache]] `UpdateSurfaceCacheAllocationState()` — カメラ距離から DesiredLockedResLevel を計算するために使用
- [[ref_lumen_scene_data]] `FLumenViewOrigin::Init()` — `CardMaxDistance` と `MaxTraceDistance` の設定
