# リファレンス：LumenSurfaceCacheFeedback.h / LumenSurfaceCacheFeedback.cpp

- グループ: a - Surface Cache
- 上位: [[a_lumen_surface_cache]]
- 関連: [[ref_lumen_scene_data]] | [[ref_lumen_surface_cache]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenSurfaceCacheFeedback.h/cpp`

---

## 概要

トレーシングシェーダーが「どの Surface Cache ページを参照したか」を GPU → CPU にフィードバックし、  
アトラスの解像度割り当てを動的に最適化する仕組み。  
**よく参照されるページは高解像度に、参照されないページは低解像度に格下げ**される。

---

## フィードバックループ全体フロー

```
[GPU] トレーシングシェーダー実行
  ↓ Surface Cache の CardPage を参照
  ↓ RWSurfaceCacheFeedbackBuffer[タイルインデックス] = CardPageIndex を書き込む
  ↓ RWSurfaceCacheFeedbackBufferAllocator でカウントアップ

[GPU → CPU] SubmitFeedbackBuffer() でリードバックを予約
  ↓ MaxReadbackBuffers = 4 個のリングバッファ方式
  ↓ 数フレーム後（GPU 処理完了後）に CPU で読み取り可能になる

[CPU] GetLatestReadbackBuffer() で利用可能なバッファを取得
  ↓ フィードバックデータを解析
  ↓ UpdateSurfaceCacheFeedback() で参照頻度集計
  ↓ 参照頻度が高い CardPage → 高解像度を割り当て
  ↓ 参照されない CardPage → 低解像度に格下げ
```

---

## namespace Lumen — フィードバックバッファサイズ計算

```cpp
namespace Lumen {
    uint32 GetFeedbackBufferTileSize();
    uint32 GetFeedbackBufferTileWrapMask();
}
```

### GetFeedbackBufferTileSize

```cpp
uint32 GetFeedbackBufferTileSize();
```

#### 戻り値
`uint32` — フィードバックバッファの 1 タイルのサイズ（ピクセル単位）。デフォルト 8 ピクセル。

#### 内部動作
```cpp
return CVarLumenSurfaceCacheFeedbackTileSize.GetValueOnRenderThread(); // 通常 8
```

#### 使用箇所
- [[ref_lumen_surface_cache_feedback]] `FLumenSurfaceCacheFeedback::AllocateFeedbackResources()` — バッファサイズ計算

---

### GetFeedbackBufferTileWrapMask

```cpp
uint32 GetFeedbackBufferTileWrapMask();
```

#### 戻り値
`uint32` — タイルインデックスのラップマスク（= `GetFeedbackBufferTileSize() - 1`）

#### 使用箇所
- シェーダー側でフィードバックバッファへの書き込み位置をラップするのに使用

---

## FLumenSurfaceCacheFeedback

> **概要**: GPU フィードバックバッファの管理クラス。`FRenderResource` を継承し、RHI リソースのライフタイムを管理する。リングバッファ方式で GPU → CPU のリードバックを遅延受け取りする。

```cpp
class FLumenSurfaceCacheFeedback : public FRenderResource {
public:
    // 内部クラス
    struct FFeedbackResources { ... };
    struct FReadbackBuffer { ... };

    // 公開メソッド
    void AllocateFeedbackResources(
        FRDGBuilder& GraphBuilder,
        FFeedbackResources& OutResources,
        const FSceneViewFamily& ViewFamily);

    FRDGBufferUAV* GetDummyFeedbackAllocatorUAV(FRDGBuilder& GraphBuilder);
    FRDGBufferUAV* GetDummyFeedbackUAV(FRDGBuilder& GraphBuilder);

    void SubmitFeedbackBuffer(
        const FViewInfo& View,
        FRDGBuilder& GraphBuilder,
        const FFeedbackResources& FeedbackResources);

    const FReadbackBuffer* GetLatestReadbackBuffer();

    FIntPoint GetFeedbackBufferTileJitter() const;
    uint32    GetFrameIndex() const;
    uint64    GetGPUSizeBytes(bool bLogSizes) const;
};
```

---

## FLumenSurfaceCacheFeedback::FFeedbackResources

フレームごとにフィードバック書き込みに使う RDG バッファの参照をまとめた内部クラス。

```cpp
struct FFeedbackResources {
    FRDGBufferUAV* BufferAllocatorUAV;  // 書き込み数カウンタの UAV
    FRDGBufferSRV* BufferAllocatorSRV;  // 同上 SRV
    FRDGBufferUAV* BufferUAV;           // フィードバックデータバッファの UAV
    FRDGBufferSRV* BufferSRV;           // 同上 SRV
    uint32 BufferSize;                  // バッファサイズ（エントリ数）
};
```

### メンバ変数

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `BufferAllocatorUAV` | `FRDGBufferUAV*` | GPU シェーダーがフィードバック書き込み数をカウントする UAV |
| `BufferAllocatorSRV` | `FRDGBufferSRV*` | 同バッファの SRV（CPU リードバック後の参照カウント確認用）|
| `BufferUAV` | `FRDGBufferUAV*` | 参照された CardPageIndex を書き込む UAV |
| `BufferSRV` | `FRDGBufferSRV*` | 同バッファの SRV |
| `BufferSize` | `uint32` | エントリ数（= タイル数、画面解像度 / TileSize^2）|

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneFrameTemporaries::SurfaceCacheFeedbackResources` に書き込まれて各パスに渡される
- [[ref_lumen_tracing_utils]] `FLumenCardTracingParameters` にシェーダーバインドパラメータとして渡される

---

## AllocateFeedbackResources

```cpp
void AllocateFeedbackResources(
    FRDGBuilder& GraphBuilder,
    FFeedbackResources& OutResources,
    const FSceneViewFamily& ViewFamily);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー（バッファ生成用）|
| `OutResources` | `FFeedbackResources&` | 出力: 生成したバッファの UAV/SRV を書き込む |
| `ViewFamily` | `const FSceneViewFamily&` | 画面解像度・プラットフォーム情報 |

### 使用箇所
- [[ref_lumen_scene]] `FDeferredShadingRenderer::UpdateLumenScene()` — フレーム開始時に `LumenFrameTemporaries.SurfaceCacheFeedbackResources` を初期化

### 内部処理フロー

1. **バッファサイズの計算**
   ```cpp
   const uint32 TileSize = Lumen::GetFeedbackBufferTileSize(); // 8
   const FIntPoint ScreenTiles = FIntPoint(
       FMath::DivideAndRoundUp(ViewFamily.FamilySizeX, TileSize),
       FMath::DivideAndRoundUp(ViewFamily.FamilySizeY, TileSize));
   const uint32 BufferSize = ScreenTiles.X * ScreenTiles.Y;
   ```

2. **フィードバックバッファの生成**
   ```cpp
   FRDGBufferRef FeedbackBuffer = GraphBuilder.CreateBuffer(
       FRDGBufferDesc::CreateStructuredDesc(sizeof(uint32) * 2, BufferSize),
       TEXT("Lumen.SurfaceCacheFeedback"));
   FRDGBufferRef AllocatorBuffer = GraphBuilder.CreateBuffer(
       FRDGBufferDesc::CreateStructuredDesc(sizeof(uint32), 1),
       TEXT("Lumen.SurfaceCacheFeedbackAllocator"));
   ```

3. **UAV / SRV の生成と書き込み**
   ```cpp
   OutResources.BufferUAV          = GraphBuilder.CreateUAV(FeedbackBuffer);
   OutResources.BufferSRV          = GraphBuilder.CreateSRV(FeedbackBuffer);
   OutResources.BufferAllocatorUAV = GraphBuilder.CreateUAV(AllocatorBuffer);
   OutResources.BufferAllocatorSRV = GraphBuilder.CreateSRV(AllocatorBuffer);
   OutResources.BufferSize         = BufferSize;
   ```

---

## SubmitFeedbackBuffer

```cpp
void SubmitFeedbackBuffer(
    const FViewInfo& View,
    FRDGBuilder& GraphBuilder,
    const FFeedbackResources& FeedbackResources);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `View` | `const FViewInfo&` | 現在のビュー（GPU マスク取得用）|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー（リードバックパス追加用）|
| `FeedbackResources` | `const FFeedbackResources&` | GPU に書き込まれたフィードバックバッファ |

### 使用箇所
- [[ref_lumen_scene]] `FDeferredShadingRenderer::UpdateLumenScene()` — GI / 反射トレース完了後、フレーム終盤で呼ばれる

### 内部処理フロー

1. **リングバッファのインデックス確認**
   ```cpp
   if (ReadbackBuffersNumPending >= MaxReadbackBuffers) {
       return; // フライト中バッファが満杯 → スキップ
   }
   ```

2. **GPU → CPU リードバックのリクエスト**
   ```cpp
   FRHIGPUBufferReadback* ReadbackBuffer = ReadbackBuffers[ReadbackBuffersWriteIndex];
   ReadbackBuffer->EnqueueCopy(GraphBuilder, FeedbackResources.Buffer, sizeof(uint32) * 2 * FeedbackResources.BufferSize);
   ReadbackBuffer->EnqueueCopy(GraphBuilder, FeedbackResources.AllocatorBuffer, sizeof(uint32));
   ```

3. **インデックスの更新**
   ```cpp
   ReadbackBuffersWriteIndex = (ReadbackBuffersWriteIndex + 1) % MaxReadbackBuffers;
   ReadbackBuffersNumPending++;
   FrameIndex++; // ジッター用フレームカウンタ
   ```

---

## GetLatestReadbackBuffer

```cpp
const FReadbackBuffer* GetLatestReadbackBuffer();
```

### 戻り値
`const FReadbackBuffer*` — CPU で読み取り可能な最新のリードバックバッファ（null = まだ準備できていない）

### 使用箇所
- [[ref_lumen_surface_cache]] `UpdateSurfaceCacheAllocationState()` — フィードバックデータを解析してページ解像度を調整

### 内部処理フロー

1. **フライト中バッファがあるか確認**
   ```cpp
   if (ReadbackBuffersNumPending == 0) return nullptr;
   ```

2. **最も古いバッファが準備できているか確認**
   ```cpp
   int32 ReadIndex = (ReadbackBuffersWriteIndex - ReadbackBuffersNumPending + MaxReadbackBuffers) % MaxReadbackBuffers;
   FRHIGPUBufferReadback* OldestReadback = ReadbackBuffers[ReadIndex];
   if (!OldestReadback->IsReady()) return nullptr;
   ```

3. **データを取得してバッファを返す**
   ```cpp
   ReadbackBuffersNumPending--;
   return OldestReadback->Lock(); // CPU 側ポインタを返す
   ```

---

> [!note]- GetDummyFeedbackAllocatorUAV / GetDummyFeedbackUAV — フィードバック無効時のダミー
> 
> ```cpp
> FRDGBufferUAV* GetDummyFeedbackAllocatorUAV(FRDGBuilder& GraphBuilder);
> FRDGBufferUAV* GetDummyFeedbackUAV(FRDGBuilder& GraphBuilder);
> ```
> 
> フィードバック機能が無効なプラットフォームや設定の場合、シェーダーバインドを壊さないためのダミー UAV を返す。実際には書き込みが行われても影響がない 1 要素バッファを使用。
> 
> **使用箇所**: [[ref_lumen_tracing_utils]] `SetupLumenCardTracingParameters()` — フィードバック無効時のバインド

> [!note]- GetFeedbackBufferTileJitter — ジッター値の取得
> 
> ```cpp
> FIntPoint GetFeedbackBufferTileJitter() const;
> ```
> 
> フレームごとにタイルオフセットをずらすことで、複数フレームにわたって全タイルのフィードバックを収集する（テンポラルカバレッジ向上）。
> 
> **戻り値**: `FIntPoint` — `{FrameIndex % TileSize, FrameIndex / TileSize % TileSize}` から計算したオフセット
> 
> **使用箇所**: [[ref_lumen_tracing_utils]] `FLumenCardTracingParameters::SurfaceCacheFeedbackBufferTileJitter` にセットされてシェーダーに渡される

---

## プライベートメンバ

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `FrameIndex` | `uint32` | フレームカウンタ（ジッター計算用）|
| `MaxReadbackBuffers` | `const int32` | フライト中バッファ数の上限（= 4）|
| `ReadbackBuffersWriteIndex` | `int32` | 現在の書き込み先バッファインデックス（0〜3）|
| `ReadbackBuffersNumPending` | `int32` | 読み取り待ちバッファ数（0〜MaxReadbackBuffers）|
| `ReadbackBuffers` | `TArray<FRHIGPUBufferReadback*>` | RHI リードバックバッファリスト（4 本）|
| `DummyFeedbackBuffer` | `FRDGPooledBuffer` | ダミー UAV 用の永続バッファ |

---

## シェーダー側のバインディング（FLumenCardTracingParameters より）

```cpp
// LumenTracingUtils.h より（フィードバック書き込み用）
SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>,   RWCardPageLastUsedBuffer)
SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>,   RWCardPageHighResLastUsedBuffer)
SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint>,   RWSurfaceCacheFeedbackBufferAllocator)
SHADER_PARAMETER_RDG_BUFFER_UAV(RWStructuredBuffer<uint2>,  RWSurfaceCacheFeedbackBuffer)

SHADER_PARAMETER(uint32,    SurfaceCacheFeedbackBufferSize)
SHADER_PARAMETER(uint32,    SurfaceCacheFeedbackBufferTileWrapMask)
SHADER_PARAMETER(FIntPoint, SurfaceCacheFeedbackBufferTileJitter)
SHADER_PARAMETER(float,     SurfaceCacheFeedbackResLevelBias)
SHADER_PARAMETER(uint32,    SurfaceCacheUpdateFrameIndex)
```
