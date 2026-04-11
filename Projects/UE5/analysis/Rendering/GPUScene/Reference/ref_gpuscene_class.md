# リファレンス：GPUScene.h / GPUScene.cpp

- グループ: Reference
- 上位: [[09_gpuscene_overview]]
- 関連: [[ref_gpuscene_params]] | [[b_gpuscene_update]]
- ソース: `Engine/Source/Runtime/Renderer/Private/GPUScene.h/.cpp`

---

## 概要

`FGPUScene` は Nanite・Lumen・VSM・RayTracing など全サブシステムが参照する  
**GPU 上の永続プリミティブ/インスタンスデータベース**。  
CPU 側の差分更新（Scatter Upload）により最小帯域で GPU バッファを維持する。

---

## FGPUScene — パブリックメンバ変数

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `PrimitiveBuffer` | `TPersistentStructuredBuffer<FVector4f>` | プリミティブシェーダーデータ（行列・バウンド等）|
| `PrimitiveUploadBuffer` | `FRDGAsyncScatterUploadBuffer` | PrimitiveBuffer の差分アップロードバッファ |
| `InstanceSceneDataBuffer` | `TPersistentStructuredBuffer<FVector4f>` | インスタンスごとの変換データ |
| `InstanceSceneUploadBuffer` | `FRDGAsyncScatterUploadBuffer` | InstanceSceneData の差分アップロードバッファ |
| `InstancePayloadDataBuffer` | `TPersistentStructuredBuffer<FVector4f>` | カスタムペイロードデータ |
| `InstancePayloadUploadBuffer` | `FRDGAsyncScatterUploadBuffer` | InstancePayloadData の差分アップロードバッファ |
| `LightmapDataBuffer` | `TPersistentStructuredBuffer<FVector4f>` | ライトマップ UV・係数 |
| `LightmapUploadBuffer` | `FRDGAsyncScatterUploadBuffer` | LightmapData の差分アップロードバッファ |
| `bUpdateAllPrimitives` | `bool` | true なら次フレームで全プリミティブを強制再アップロード |

---

## FGPUScene — プライベートメンバ変数

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `Scene` | `FScene&` | オーナーシーンへの参照 |
| `InstanceSceneDataAllocator` | `FSpanAllocator` | インスタンスデータスロット管理 |
| `LightmapDataAllocator` | `FSpanAllocator` | ライトマップデータスロット管理 |
| `InstancePayloadDataAllocator` | `FSpanAllocator` | ペイロードデータスロット管理 |
| `PrimitivesToUpdate` | `TArray<FPersistentPrimitiveIndex>` | 次 Update で処理するプリミティブ一覧 |
| `PrimitiveDirtyState` | `TArray<EPrimitiveDirtyState>` | プリミティブごとのダーティフラグ |
| `LightDataBuffer` | `TPersistentByteAddressBuffer<FLightSceneData>` | ライトシーンデータ永続バッファ |
| `NumGPULights` | `int32` | GPU に登録済みのライト数 |
| `SceneFrameNumber` | `uint32` | 最後に Update された時のフレーム番号 |
| `DynamicPrimitivesOffset` | `int32` | 動的プリミティブの先頭インデックス |
| `bIsEnabled` | `bool` | GPUScene 有効か（SM5以上で true）|
| `bInBeginEndBlock` | `bool` | BeginRender～EndRender ブロック中か |
| `CurrentDynamicContext` | `FGPUSceneDynamicContext*` | 現在の動的コンテキスト |
| `FeatureLevel` | `ERHIFeatureLevel::Type` | 現在のフィーチャーレベル |
| `CachedRegisteredBuffers` | `FRegisteredBuffers` | フレーム中キャッシュされた RDG バッファ参照 |
| `InstanceRangesToClear` | `TArray<FInstanceRange>` | 次 Update でクリアすべきインスタンス範囲 |

---

## FRegisteredBuffers — フレーム中のバッファキャッシュ

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `PrimitiveBuffer` | `FRDGBufferRef` | RDG 上の PrimitiveBuffer 参照 |
| `InstanceSceneDataBuffer` | `FRDGBufferRef` | RDG 上の InstanceSceneDataBuffer 参照 |
| `InstancePayloadDataBuffer` | `FRDGBufferRef` | RDG 上の InstancePayloadDataBuffer 参照 |
| `LightmapDataBuffer` | `FRDGBufferRef` | RDG 上の LightmapDataBuffer 参照 |
| `LightDataBuffer` | `FRDGBufferRef` | RDG 上の LightDataBuffer 参照 |

---

## 主要関数

| 関数 | ファイル:行 | 説明 |
|------|-----------|------|
| `BeginRender()` | `GPUScene.cpp:726` | フレーム開始・動的スロット確定・バッファ登録 |
| `EndRender()` | `GPUScene.cpp:746` | 動的スロットリセット・コンテキストクリア |
| `Update()` | `GPUScene.cpp:1978` | 差分 Scatter Upload の実行 |
| `UpdateInternal()` | `GPUScene.cpp:883` | 実際の upload 処理・SceneUB 更新 |
| `UpdateGPULights()` | `GPUScene.cpp:765` | ライトデータの差分アップロード |
| `UploadDynamicPrimitiveShaderDataForView()` | `GPUScene.cpp:1993` | ビュー動的プリミティブ upload |
| `AddPrimitiveToUpdate()` | `GPUScene.cpp:1951` | 変更プリミティブをキューに追加 |
| `AllocateInstanceSceneDataSlots()` | `GPUScene.cpp:2019` | インスタンスデータスロット確保 |
| `FreeInstanceSceneDataSlots()` | `GPUScene.cpp:2040` | インスタンスデータスロット解放 |
| `GetShaderParameters()` | `GPUScene.h:301` | FGPUSceneResourceParameters を取得 |
| `FillSceneUniformBuffer()` | `GPUScene.h:261` | SceneUB の GPU バッファ参照を更新 |

---

> [!note]- FGPUSceneScopeBeginEndHelper による RAII 管理
> `FGPUSceneScopeBeginEndHelper` は `BeginRender()` と `EndRender()` を RAII で管理するラッパー。  
> `DeferredShadingRenderer.cpp:1799` でスタック変数として生成され、  
> スコープを外れると自動的に `EndRender()` が呼ばれる。  
> これにより `EndRender()` の呼び忘れや例外パスでのリソースリークを防いでいる。

> [!note]- PrimitivesToUpdate と EPrimitiveDirtyState の累積
> `AddPrimitiveToUpdate()` は同じプリミティブを複数回呼ばれても  
> `PrimitiveDirtyState` にダーティフラグをビットOR で累積する。  
> `Update()` 時に一度だけ処理されるため、1フレーム中に複数回の変化が来ても安全。  
> フラグは `EPrimitiveDirtyState::ChangedAll`（全変化）/ `ChangedTransform`（変形のみ）等に分かれる。

> [!note]- bIsEnabled とフィーチャーレベルの関係
> `FGPUScene::SetEnabled()` は SM5 以上（Desktop High）の場合のみ `bIsEnabled = true` にする。  
> Mobile（ES3.1）や低スペック環境では GPUScene が無効になり、  
> 従来の UBO ベースのプリミティブデータ送信にフォールバックする。  
> `bIsEnabled = false` の場合、`Update()` / `BeginRender()` などは早期 return する。
