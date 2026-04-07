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

従来の CPU 側のカリングに比べて、大量プリミティブ環境でのパフォーマンスを改善する。

---

## FLumenSceneReadback : public FRenderResource

GPU から CPU へのリードバックバッファ管理クラス。  
`FLumenSurfaceCacheFeedback` と同様のリングバッファ方式。

### 内部構造体

```cpp
struct FAddOp {
    uint32 PrimitiveGroupIndex;  // 追加対象のプリミティブグループインデックス
    float  DistanceSq;           // カメラからの距離の2乗
};

struct FRemoveOp {
    uint32 PrimitiveGroupIndex;  // 削除対象のプリミティブグループインデックス
};

struct FBuffersRHI {
    FRHIGPUBufferReadback* AddOps;    // 追加操作リードバックバッファ
    FRHIGPUBufferReadback* RemoveOps; // 削除操作リードバックバッファ
};

struct FBuffersRDG {
    FRDGBufferRef AddOps;    // RDG 書き込み用バッファ
    FRDGBufferRef RemoveOps; // RDG 書き込み用バッファ
};
```

### メソッド一覧

| メソッド | 説明 |
|---------|------|
| `GetWriteBuffers(GraphBuilder)` | GPU 書き込み用 RDG バッファを取得（または新規生成）|
| `SubmitWriteBuffers(GraphBuilder, BuffersRDG)` | 書き込んだバッファをリードバックキューに投入 |
| `GetLatestReadbackBuffers()` | CPU 読み取り可能な最新リードバックバッファを返す |
| `GetMaxAddOps()` | 1 フレームの最大 Add 操作数（= 1024）|
| `GetMaxRemoveOps()` | 1 フレームの最大 Remove 操作数（= 1024）|
| `GetAddOpsBufferSizeInBytes()` | Add バッファのバイトサイズ |
| `GetRemoveOpsBufferSizeInBytes()` | Remove バッファのバイトサイズ |

### プライベートメンバ

```cpp
const int32 MaxAddOps = 1024;
const int32 MaxRemoveOps = 1024;
const int32 MaxReadbackBuffers = 4;     // フライト中リードバック数
int32 ReadbackBuffersWriteIndex = 0;
int32 ReadbackBuffersNumPending = 0;
TArray<FBuffersRHI> ReadbackBuffers;
```

---

## LumenScene 名前空間

```cpp
namespace LumenScene {
    // Card の最大有効距離を返す（距離に応じた LOD 切り替え用）
    float GetCardMaxDistance(const FViewInfo& View);

    // Card テクセル密度を返す（近距離）
    float GetCardTexelDensity();

    // 遠距離フィールド用 Card テクセル密度を返す
    float GetFarFieldCardTexelDensity();

    // 遠距離フィールド Card の最大距離を返す
    float GetFarFieldCardMaxDistance();

    // Card の最小解像度を返す（正投影カメラかどうかで異なる）
    int32 GetCardMinResolution(bool bOrthographicCamera);

    // GPU Driven な Lumen シーン更新を実行
    // - GPU コンピュートでプリミティブの可視性判定
    // - AddOps / RemoveOps を GPU バッファに書き込み
    // - SubmitWriteBuffers() でリードバックキューに投入
    void GPUDrivenUpdate(
        FRDGBuilder& GraphBuilder,
        const FScene* Scene,
        TArray<FViewInfo>& Views,
        const FLumenSceneFrameTemporaries& FrameTemporaries);
}
```

---

## GPU Driven Update の処理フロー

```
LumenScene::GPUDrivenUpdate()
  │
  ├─ [GPU Compute] プリミティブのワールド AABB をフラスタムと比較
  │     → 可視なプリミティブグループを AddOps バッファに書き込み
  │     → 不可視になったグループを RemoveOps バッファに書き込み
  │
  ├─ SubmitWriteBuffers() → リードバックキューに投入
  │
  └─ [CPU, 数フレーム後] GetLatestReadbackBuffers()
        → AddOps を読み取り → FLumenSceneData に新規プリミティブを追加
        → RemoveOps を読み取り → FLumenSceneData からプリミティブを削除
```

---

## 従来方式との比較

| 比較項目 | CPU Driven（旧）| GPU Driven（新）|
|---------|----------------|----------------|
| 可視性判定 | CPU で全プリミティブをループ | GPU コンピュートで並列処理 |
| 更新レイテンシ | 即時 | 数フレーム遅延（リードバック待ち）|
| 大量プリミティブ | CPU ボトルネックになりやすい | GPU で高速処理 |
| r.LumenScene.ParallelUpdate | マルチスレッドCPU処理 | 上記の GPU 版 |
