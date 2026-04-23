# GPUScene 全体概要

- 取得日: 2026-04-10
- 対象: `D:\UnrealEngine\Engine\Source\Runtime\Renderer\Private\GPUScene.h/.cpp`
- 上位: [[01_rendering_overview]]

---

## GPUScene とは

**シーン内の全プリミティブ・インスタンスデータを GPU バッファに保持する**システム。  
Nanite・Lumen・VSM・RayTracing など全てのレンダリングサブシステムが  
このバッファを共通の真実源（Source of Truth）として参照する。

| 従来の問題 | GPUScene の解法 |
|-----------|--------------|
| Draw Call ごとに CPU からデータを送っていた | 永続 GPU バッファを差分更新（Scatter Upload） |
| GPU Driven レンダリングができなかった | GPU 上でプリミティブデータを直接参照 |
| インスタンス情報の取得がシェーダーから難しかった | `GPUSceneInstanceSceneData` を SRV で提供 |

---

## 全体アーキテクチャ

```mermaid
graph TD
    subgraph CPU["CPU（ゲームスレッド/レンダースレッド）"]
        PS[FPrimitiveSceneInfo<br>プリミティブ追加/削除/変形]
        UP[ScatterUpload<br>差分データをキューに積む]
        PS --> UP
    end

    subgraph GPUBuf["GPU バッファ（永続）"]
        PD[GPUScenePrimitiveSceneData<br>プリミティブの行列・バウンド等]
        ID[GPUSceneInstanceSceneData<br>インスタンスごとのローカル→ワールド変換]
        IP[GPUSceneInstancePayloadData<br>カスタムペイロード]
        LM[GPUSceneLightmapData<br>ライトマップ UV]
        LD[GPUSceneLightData<br>ライトデータ]
    end

    subgraph Consumers["参照先（全サブシステム）"]
        Nanite[Nanite CullRaster]
        Lumen[Lumen Tracing]
        VSM[VirtualShadowMaps]
        RT[RayTracing]
        Mesh[通常メッシュ DrawCall]
    end

    UP --> PD & ID & IP & LM & LD
    PD & ID --> Nanite & Lumen & VSM & RT & Mesh
```

---

## フレームの流れ（概略）

```
[A] FGPUScene::BeginRender()
    → アップロードバッファをロック（Scatter Upload の開始）

[B] プリミティブ変化の適用
    → Add/Remove/Update された FPrimitiveSceneInfo をバッファに差分書き込み
    → FGPUScenePrimitiveCollector::Commit() でダイナミックプリミティブも追加

[C] FGPUScene::EndRender()
    → GPU にアップロード実行（コンピュートシェーダーで Scatter）
    → SceneUniformBuffer を通じて全シェーダーへバインド

[D] 各サブシステムが FGPUSceneResourceParameters 経由で参照
    → GPUSceneInstanceSceneData / GPUScenePrimitiveSceneData を SRV で読み取り
```

---

## 主要クラス・構造体

```cpp
// GPU バッファの SRV（シェーダーバインド用）
BEGIN_SHADER_PARAMETER_STRUCT(FGPUSceneResourceParameters, )
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, GPUSceneInstanceSceneData)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, GPUSceneInstancePayloadData)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, GPUScenePrimitiveSceneData)
    SHADER_PARAMETER_RDG_BUFFER_SRV(StructuredBuffer<float4>, GPUSceneLightmapData)
    SHADER_PARAMETER_RDG_BUFFER_SRV(ByteAddressBuffer,        GPUSceneLightData)
    // 共通パラメータ
    SHADER_PARAMETER(uint32, GPUSceneInstanceDataSOAStride)
    SHADER_PARAMETER(uint32, GPUSceneFrameNumber)
    SHADER_PARAMETER(int32,  GPUSceneMaxAllocatedInstanceId)
    SHADER_PARAMETER(int32,  GPUSceneMaxPersistentPrimitiveIndex)
END_SHADER_PARAMETER_STRUCT()

// ダイナミックプリミティブの追加（InitViews 中に使用）
class FGPUScenePrimitiveCollector
{
    // プリミティブデータを登録
    void Add(
        const FMeshBatchDynamicPrimitiveData* MeshBatchData,
        const FPrimitiveUniformShaderParameters& Params,
        uint32 NumInstances,
        uint32& OutPrimitiveIndex,
        uint32& OutInstanceSceneDataOffset);

    // GPU バッファへのアップロードをキュー（BeginRender ブロック内で呼ぶ）
    void Commit();

    // コミット後にのみ有効
    const TRange<int32>& GetPrimitiveIdRange() const;
    int32 GetInstanceSceneDataOffset() const;
};
```

---

## 主要 CVar 一覧

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.GPUScene.UploadEveryFrame` | 0 | 毎フレーム全データを強制アップロード（デバッグ用） |
| `r.GPUScene.ParallelUpdate` | 1 | 並列タスクで更新 |
| `r.GPUScene.MaxPooledUploadBufferSize` | 256000 | アップロードバッファのプールサイズ上限（bytes） |

---

## コード実行フロー

### エントリポイント

```
FDeferredShadingSceneRenderer::Render()                         DeferredShadingRenderer.cpp
  │
  ├─[A] FGPUSceneScopeBeginEndHelper (RAII ctor)               :1799
  │      └─ FGPUScene::BeginRender()                           GPUScene.cpp:726
  │           ├─ DynamicPrimitivesOffset をリセット
  │           └─ RegisterBuffers() → CachedRegisteredBuffers
  │
  ├─[B] FScene::UpdateAllPrimitives()                          RendererScene.cpp
  │      └─ FGPUScene::Update()                               RendererScene.cpp:6541
  │           └─ UpdateInternal()                             GPUScene.cpp:883
  │                ├─ PrimitivesToUpdate をイテレート
  │                ├─ ScatterUpload バッファに差分データ書き込み
  │                └─ FillSceneUniformBuffer() → SceneUB 更新
  │
  ├─[C] UpdateGPULights()                                      GPUScene.cpp:765
  │
  ├─[D] UploadDynamicPrimitiveShaderDataForView()              DeferredShadingRenderer.cpp:2172
  │      └─ View.DynamicPrimitiveCollector を GPU 動的領域へ upload
  │
  └─[E] FGPUSceneScopeBeginEndHelper (RAII dtor)
         └─ FGPUScene::EndRender()                            GPUScene.cpp:746
              └─ DynamicPrimitivesOffset / CachedRegisteredBuffers をリセット
```

### フロー詳細

1. **[A] BeginRender** (`GPUScene.cpp:726`) — RAII スコープ開始。動的プリミティブ領域の先頭インデックスを確定し、フレーム用 RDG バッファを登録する
2. **[B] Update / UpdateInternal** (`GPUScene.cpp:1978/883`) — `RendererScene.cpp:6541` から呼ばれ、変化プリミティブのデータを Scatter Upload バッファに書き込み CS でGPU バッファへ scatter する
3. **[C] UpdateGPULights** (`GPUScene.cpp:765`) — ライト変化の差分のみ ByteAddressBuffer に upload する（オプションで非同期タスク）
4. **[D] UploadDynamicPrimitiveShaderDataForView** (`GPUScene.cpp:1993`) — `InitViews` 中に登録された動的プリミティブを GPU バッファの動的領域へ upload する
5. **[E] EndRender** (`GPUScene.cpp:746`) — RAII スコープ終了。動的スロットをリセットし次フレーム用に領域を空ける

### 関与クラス・関数一覧

| クラス/関数 | ファイル:行 | 役割 |
|------------|-----------|------|
| `FGPUSceneScopeBeginEndHelper` | `GPUScene.h:513` | BeginRender/EndRender の RAII ラッパー |
| `FGPUScene::BeginRender()` | `GPUScene.cpp:726` | フレーム開始処理 |
| `FGPUScene::Update()` | `GPUScene.cpp:1978` | 差分 Scatter Upload のエントリ |
| `FGPUScene::UpdateInternal()` | `GPUScene.cpp:883` | 実際の upload・SceneUB 更新 |
| `FGPUScene::UpdateGPULights()` | `GPUScene.cpp:765` | ライトデータ差分 upload |
| `FGPUScene::UploadDynamicPrimitiveShaderDataForView()` | `GPUScene.cpp:1993` | 動的プリミティブ upload |
| `FGPUScene::EndRender()` | `GPUScene.cpp:746` | フレーム終了・動的スロット解放 |
| `FGPUScenePrimitiveCollector` | `GPUScene.h` | 動的プリミティブの収集・登録 |
| `FSpanAllocator` | `SpanAllocator.h` | インスタンスデータスロット管理 |

---

## 主要ソースファイル一覧

| ファイル | 役割 |
|---------|------|
| `GPUScene.h` | FGPUSceneResourceParameters / FGPUScenePrimitiveCollector / シェーダーバインド定義 |
| `GPUScene.cpp` | Scatter Upload 実装・BeginRender/EndRender・並列更新タスク |
