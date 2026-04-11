# GPUScene 動的プリミティブ

- グループ: c - Dynamic
- 上位: [[09_gpuscene_overview]]
- 関連: [[b_gpuscene_update]] | [[ref_gpuscene_class]]
- ソース: `Engine/Source/Runtime/Renderer/Private/GPUScene.h/.cpp`

---

## 概要

`FGPUScenePrimitiveCollector` を使って、フレームごとに生成される**動的プリミティブ**  
（パーティクル・デカール・手続き的メッシュ等）を GPUScene バッファのダイナミック領域に登録する仕組み。  
永続プリミティブ（`Scene.Primitives`）とは異なり、次フレームには消える一時データ。

---

## 動的プリミティブとは

| 種類 | 例 |
|------|-----|
| パーティクル | Niagara メッシュ パーティクル |
| デカール | DBuffer デカール |
| 手続き的メッシュ | ランタイム生成メッシュ |
| エディタ補助オブジェクト | ワイヤーフレーム・バウンド表示 |

---

## FGPUScenePrimitiveCollector — API

```cpp
class FGPUScenePrimitiveCollector
{
    // プリミティブデータを登録
    // OutPrimitiveIndex / OutInstanceSceneDataOffset は Commit() 後に有効になる
    void Add(
        const FMeshBatchDynamicPrimitiveData* MeshBatchData,
        const FPrimitiveUniformShaderParameters& Params,
        uint32 NumInstances,
        uint32& OutPrimitiveIndex,
        uint32& OutInstanceSceneDataOffset);

    // GPU バッファへのアップロードをキュー（BeginRender ブロック内で呼ぶ）
    void Commit();

    // Commit() 後にのみ有効
    const TRange<int32>& GetPrimitiveIdRange() const;
    int32 GetInstanceSceneDataOffset() const;
};
```

---

## コード実行フロー

### エントリポイント

```
FDeferredShadingSceneRenderer::InitViews()
  ├─ FSceneRenderer::SetupDynamicPrimitiveCollector()
  │    └─ FGPUScenePrimitiveCollector::Commit()
  │
  └─ FGPUScene::UploadDynamicPrimitiveShaderDataForView()   GPUScene.cpp:1993
       └─ UploadDynamicPrimitiveShaderDataForViewInternal() GPUScene.cpp:1782
            ├─ DynamicPrimitivesOffset 以降の領域に upload
            └─ FRDGAsyncScatterUploadBuffer でバッファ更新
```

### フロー詳細

1. `InitViews()` 中に各プロキシが `FGPUScenePrimitiveCollector::Add()` を呼ぶ
2. `Add()` は登録データをローカルに蓄積する（この時点では GPU バッファ確保なし）
3. `Commit()` でデータを確定し、`DynamicPrimitivesOffset` から連続スロットを確保する
4. `Commit()` 後に `OutPrimitiveIndex` / `OutInstanceSceneDataOffset` が確定する
5. `UploadDynamicPrimitiveShaderDataForView()` で GPU バッファの動的領域に scatter upload する
6. フレーム終了時（`EndRender()`）に `DynamicPrimitivesOffset` がリセットされ動的スロットが解放される

### 関与クラス・関数一覧

| クラス/関数 | ファイル:行 | 役割 |
|------------|-----------|------|
| `FGPUScenePrimitiveCollector::Add()` | `GPUScene.h` | 動的プリミティブデータを登録 |
| `FGPUScenePrimitiveCollector::Commit()` | `GPUScene.h` | データ確定・スロット確保 |
| `FGPUScene::UploadDynamicPrimitiveShaderDataForView()` | `GPUScene.cpp:1993` | ビュー動的プリミティブを upload |
| `UploadDynamicPrimitiveShaderDataForViewInternal()` | `GPUScene.cpp:1782` | 実際の upload 処理 |
| `FGPUSceneDynamicContext` | `GPUScene.h` | 動的コンテキスト（BeginRender で受け渡し）|

---

## 動的プリミティブの GPU バッファ上の位置

```
GPUScenePrimitiveSceneData バッファ:
  [0 .. MaxPersistentPrimitiveIndex-1]   ← 永続プリミティブ（Scene.Primitives）
  [MaxPersistentPrimitiveIndex .. N-1]   ← 動的プリミティブ（毎フレームリセット）
                                           ↑ DynamicPrimitivesOffset
```

シェーダーは `GPUSceneInstanceSceneData[instanceIndex].PrimitiveId` でインデックスを取得し、  
プリミティブが永続か動的かを区別せずに `GPUScenePrimitiveSceneData[primId]` を参照できる。

---

## 関連リファレンス

- [[ref_gpuscene_class]] — FGPUScene メンバ変数・メソッド一覧
- [[ref_gpuscene_params]] — FGPUSceneResourceParameters シェーダーバインド定義
- [[b_gpuscene_update]] — BeginRender/Update/EndRender フロー
