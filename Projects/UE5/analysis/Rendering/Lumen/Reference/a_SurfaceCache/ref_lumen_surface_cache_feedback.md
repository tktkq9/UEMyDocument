# リファレンス：LumenSurfaceCacheFeedback.h / LumenSurfaceCacheFeedback.cpp

- グループ: a - Surface Cache
- 上位: [[a_lumen_surface_cache]]
- 関連: [[ref_lumen_scene_data]] | [[ref_lumen_surface_cache]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSurfaceCacheFeedback.h/cpp`

---

## 概要

トレーシングシェーダーが「どの Surface Cache ページを参照したか」を GPU→CPU にフィードバックし、  
アトラスの解像度割り当てを動的に最適化する仕組み。  
**よく参照されるページは高解像度に、参照されないページは低解像度に格下げ**される。

---

## Lumen 名前空間（フィードバックバッファサイズ計算）

```cpp
namespace Lumen {
    // フィードバックバッファのタイルサイズを返す
    uint32 GetFeedbackBufferTileSize();

    // タイルインデックスのラップマスクを返す（タイルサイズ - 1）
    uint32 GetFeedbackBufferTileWrapMask();
}
```

---

## FLumenSurfaceCacheFeedback : public FRenderResource

GPU フィードバックバッファの管理クラス。`FRenderResource` を継承し、RHI リソースのライフタイムを管理。

### 内部クラス: FFeedbackResources

```cpp
class FFeedbackResources {
    FRDGBufferUAV* BufferAllocatorUAV;  // フィードバック書き込み数カウンタ（UAV）
    FRDGBufferSRV* BufferAllocatorSRV;  // 同上（SRV）
    FRDGBufferUAV* BufferUAV;           // フィードバックデータバッファ（UAV）
    FRDGBufferSRV* BufferSRV;           // 同上（SRV）
    uint32 BufferSize;                  // バッファサイズ（エントリ数）
};
```

### メソッド一覧

| メソッド | 説明 |
|---------|------|
| `AllocateFeedbackResources(GraphBuilder, Resources, ViewFamily)` | フレームごとにフィードバック用 RDG バッファを確保 |
| `GetDummyFeedbackAllocatorUAV(GraphBuilder)` | フィードバック無効時のダミー UAV を返す |
| `GetDummyFeedbackUAV(GraphBuilder)` | フィードバック無効時のダミーデータ UAV を返す |
| `SubmitFeedbackBuffer(View, GraphBuilder, FeedbackResources)` | GPU→CPU リードバックを予約する |
| `GetLatestReadbackBuffer()` | 最新の読み取り可能なリードバックバッファを返す（null の可能性あり）|
| `GetFeedbackBufferTileJitter()` | ジッター値（フレームごとにオフセットを変えてカバレッジを上げる）|
| `GetFrameIndex()` | 現在のフレームインデックスを返す |
| `GetGPUSizeBytes(bool)` | GPU メモリ使用量を返す |

### プライベートメンバ

```cpp
uint32 FrameIndex = 0;                  // フレームカウンタ（ジッター用）
const int32 MaxReadbackBuffers = 4;     // リードバックバッファ数（フライト中バッファ数）
int32 ReadbackBuffersWriteIndex = 0;    // 現在の書き込み先インデックス
int32 ReadbackBuffersNumPending = 0;    // 読み取り待ちバッファ数
TArray<FRHIGPUBufferReadback*> ReadbackBuffers; // RHI リードバックバッファリスト
```

---

## フィードバックループの全体フロー

```
[GPU] トレーシングシェーダー実行
  ↓ Surface Cache の CardPage を参照
  ↓ RWSurfaceCacheFeedbackBuffer[タイルインデックス] = CardPageIndex を書き込む
  ↓ RWSurfaceCacheFeedbackBufferAllocator でカウントアップ

[GPU→CPU] SubmitFeedbackBuffer() でリードバックを予約
  ↓ MaxReadbackBuffers = 4 個のリングバッファ方式
  ↓ 数フレーム後に CPU で読み取り可能になる

[CPU] GetLatestReadbackBuffer() で利用可能なバッファを取得
  ↓ フィードバックデータを解析
  ↓ 参照頻度が高い CardPage → 高解像度を割り当て
  ↓ 参照されない CardPage → 低解像度に格下げ
```

---

## シェーダー側のバインディング（FLumenCardTracingParameters より）

```cpp
// LumenTracingUtils.h より（フィードバック書き込み用）
SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>, RWCardPageLastUsedBuffer)
SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>, RWCardPageHighResLastUsedBuffer)
SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>, RWSurfaceCacheFeedbackBufferAllocator)
SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint2>, RWSurfaceCacheFeedbackBuffer)

SHADER_PARAMETER(uint32, SurfaceCacheFeedbackBufferSize)         // バッファサイズ
SHADER_PARAMETER(uint32, SurfaceCacheFeedbackBufferTileWrapMask) // ラップマスク
SHADER_PARAMETER(FIntPoint, SurfaceCacheFeedbackBufferTileJitter) // フレームジッター
SHADER_PARAMETER(float, SurfaceCacheFeedbackResLevelBias)        // 解像度レベルバイアス
SHADER_PARAMETER(uint32, SurfaceCacheUpdateFrameIndex)           // フレームインデックス
```
