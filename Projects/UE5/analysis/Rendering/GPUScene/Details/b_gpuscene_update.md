# GPUScene 更新フロー

- グループ: b - Update
- 上位: [[09_gpuscene_overview]]
- 関連: [[a_gpuscene_buffers]] | [[c_gpuscene_dynamic]] | [[ref_gpuscene_class]]
- ソース: `Engine/Source/Runtime/Renderer/Private/GPUScene.h/.cpp`

---

## 概要

フレームごとの GPUScene 更新フロー。  
`FGPUSceneScopeBeginEndHelper`（RAII）が `BeginRender` / `EndRender` のペアを管理し、  
その間に `Update()` → `UpdateInternal()` で差分データを GPU へ Scatter Upload する。

---

## フレーム更新の全体像

```
FDeferredShadingSceneRenderer::Render()
  │
  ├─[A] FGPUSceneScopeBeginEndHelper (ctor)        DeferredShadingRenderer.cpp:1799
  │      └─ FGPUScene::BeginRender()               GPUScene.cpp:726
  │           ├─ DynamicPrimitivesOffset をリセット
  │           ├─ CurrentDynamicContext をセット
  │           └─ RegisterBuffers() → CachedRegisteredBuffers
  │
  ├─[B] FScene::UpdateAllPrimitives()              RendererScene.cpp:6534
  │      └─ FGPUScene::Update()                   RendererScene.cpp:6541
  │           └─ UpdateInternal()                 GPUScene.cpp:883
  │                ├─ 全 PrimitivesToUpdate をイテレート
  │                ├─ ScatterUpload バッファに差分データ書き込み
  │                ├─ Compute Shader で GPU 永続バッファへ scatter
  │                └─ FillSceneUniformBuffer() → SceneUB を更新
  │
  ├─[C] UpdateGPULights()                         GPUScene.cpp:765
  │      └─ TAsyncByteAddressBufferScatterUploader でライトデータを upload
  │
  ├─[D] UploadDynamicPrimitiveShaderDataForView()  DeferredShadingRenderer.cpp:2172
  │      └─ UploadDynamicPrimitiveShaderDataForViewInternal()
  │           └─ View の DynamicPrimitiveCollector をアップロード
  │
  └─[E] FGPUSceneScopeBeginEndHelper (dtor)
         └─ FGPUScene::EndRender()                GPUScene.cpp:746
              ├─ DynamicPrimitivesOffset をリセット
              ├─ CurrentDynamicContext をクリア
              └─ CachedRegisteredBuffers をクリア
```

---

## 各ステップの詳細

### [A] BeginRender (`GPUScene.cpp:726`)

- `bInBeginEndBlock = true` にしてアサーションガードを設定
- `DynamicPrimitivesOffset = Scene.GetMaxPersistentPrimitiveIndex()` — 動的プリミティブの開始インデックスを確定
- `RegisterBuffers()` で現フレームの RDG バッファ参照を `CachedRegisteredBuffers` にキャッシュ

### [B] UpdateInternal (`GPUScene.cpp:883`)

- `r.GPUScene.UploadEveryFrame = 1` または `bUpdateAllPrimitives = true` の場合、全プリミティブを強制更新
- `PrimitivesToUpdate` のリストから変更されたプリミティブを処理
- 並列タスクで `FPrimitiveSceneShaderData` を構築し、scatter upload バッファに書き込む
- CS ディスパッチで GPU 側バッファを更新
- `FillSceneUniformBuffer()` で SceneUniformBuffer の SRV を最新のバッファに向ける

### [C] UpdateGPULights (`GPUScene.cpp:765`)

- `FLightSceneChangeSet` の差分のみを `TAsyncByteAddressBufferScatterUploader` でアップロード
- `r.GPUScene.Lights.AsyncSetup = 1` の場合、非同期タスクでデータ準備

### [D] UploadDynamicPrimitiveShaderDataForView (`GPUScene.cpp:1993`)

- `View.DynamicPrimitiveCollector` に積まれた動的プリミティブを GPU バッファのダイナミック領域へアップロード
- `DynamicPrimitivesOffset` 以降の領域を使用（永続プリミティブ領域を汚染しない）

### [E] EndRender (`GPUScene.cpp:746`)

- `DynamicPrimitivesOffset` を `Scene.GetMaxPersistentPrimitiveIndex()` にリセット（動的スロット解放）
- `CachedRegisteredBuffers = {}` でフレーム一時のバッファ参照をクリア

---

## AddPrimitiveToUpdate の呼び出しタイミング

`AddPrimitiveToUpdate()` は `Update()` の**前**に呼ばれ、変化プリミティブをキューイングする：

| 呼び出し元 | タイミング |
|-----------|----------|
| `FScene::AddPrimitive()` | プリミティブ追加時 |
| `FScene::UpdatePrimitive()` | 変形・マテリアル変化時 |
| `FScene::RemovePrimitive()` | プリミティブ削除時 |
| `RendererScene.cpp:6519` | UniformBuffer 更新時 |

---

## 関連リファレンス

- [[ref_gpuscene_class]] — FGPUScene メンバ変数・メソッド一覧
- [[a_gpuscene_buffers]] — バッファ構造と FSpanAllocator
- [[c_gpuscene_dynamic]] — 動的プリミティブの登録フロー
