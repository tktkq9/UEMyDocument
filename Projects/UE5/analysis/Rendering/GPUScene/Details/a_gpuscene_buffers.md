# GPUScene バッファ構造

- グループ: a - Buffers
- 上位: [[09_gpuscene_overview]]
- 関連: [[b_gpuscene_update]] | [[ref_gpuscene_class]]
- ソース: `Engine/Source/Runtime/Renderer/Private/GPUScene.h/.cpp`

---

## 概要

GPUScene が保持する**永続 GPU バッファ**の種類と役割を説明する。  
全てのバッファは `FRDGAsyncScatterUploadBuffer` を通じて差分更新（Scatter Upload）される。  
シェーダーからは `FGPUSceneResourceParameters` の SRV として参照される。

---

## バッファ一覧

| バッファ名 | 格納データ | 更新タイミング |
|-----------|-----------|--------------|
| `PrimitiveBuffer` | `FPrimitiveSceneShaderData`（行列・バウンド等） | プリミティブ追加/変形時 |
| `InstanceSceneDataBuffer` | `FInstanceSceneShaderData`（インスタンス変換） | インスタンス追加/変形時 |
| `InstancePayloadDataBuffer` | カスタムペイロード（float4配列） | プリミティブ依存 |
| `LightmapDataBuffer` | ライトマップ UV / 係数 | ライトマップ変化時 |
| `LightDataBuffer` | `FLightSceneData`（ライト全種） | ライト追加/変化時 |

---

## FRDGAsyncScatterUploadBuffer — Scatter Upload の仕組み

```
[CPU側]
  PrimitivesToUpdate に変化プリミティブ ID を積む
    ↓
  UpdateInternal() で upload buffer に差分データを書き込む
    ↓
  Compute Shader が scatter → 永続バッファの該当スロットに書き込む

[GPU側の永続バッファ]
  GPUScenePrimitiveSceneData[primitiveIndex]   ← PrimitiveBuffer
  GPUSceneInstanceSceneData[instanceOffset+i]  ← InstanceSceneDataBuffer
```

- 全データを毎フレーム送らないため帯域・コストを最小化できる
- `r.GPUScene.UploadEveryFrame = 1` で強制フルアップロード（デバッグ用）

---

## FSpanAllocator — インスタンスデータのスロット管理

インスタンスデータは**可変長スロット**（プリミティブごとにインスタンス数が異なる）のため、  
`FSpanAllocator` がスロット範囲を管理する。

```cpp
// プリミティブ追加時
int32 Offset = InstanceSceneDataAllocator.Allocate(NumInstances);
// → InstanceSceneDataBuffer[Offset .. Offset+NumInstances-1] が確保される

// プリミティブ削除時
InstanceSceneDataAllocator.Free(Offset, NumInstances);
// → スロットが解放されて次の Allocate で再利用可能になる
```

| 関数 | 説明 |
|------|------|
| `AllocateInstanceSceneDataSlots()` | インスタンスデータスロットを確保 |
| `FreeInstanceSceneDataSlots()` | スロットを解放 |
| `AllocateInstancePayloadDataSlots()` | ペイロードスロットを確保 |
| `FreeInstancePayloadDataSlots()` | ペイロードスロットを解放 |

---

## コード実行フロー

### エントリポイント（バッファ割り当て）

```
FScene::AddPrimitive()
  └─ FGPUScene::AllocateInstanceSceneDataSlots()   GPUScene.cpp:2019
       └─ InstanceSceneDataAllocator.Allocate()    FSpanAllocator
            └─ AddOrMergeInstanceRange()            InstanceRangesToClear に積む
```

### フロー詳細

1. `FScene::AddPrimitive()` がプリミティブを登録し、インスタンス数を確定する
2. `AllocateInstanceSceneDataSlots(PrimIndex, NumInstances)` がスロット範囲を割り当てる
3. `InstanceRangesToClear` に「クリアすべき範囲」として記録する
4. `AddPrimitiveToUpdate()` で `PrimitivesToUpdate` に追加し、次の `Update()` を待つ
5. `Update()` → `UpdateInternal()` で scatter upload バッファに書き込み、GPU に送る

### 関与クラス・関数一覧

| クラス/関数 | ファイル:行 | 役割 |
|------------|-----------|------|
| `FSpanAllocator` | `SpanAllocator.h` | スロット範囲を管理 |
| `AllocateInstanceSceneDataSlots()` | `GPUScene.cpp:2019` | インスタンスデータ割り当て |
| `FreeInstanceSceneDataSlots()` | `GPUScene.cpp:2040` | インスタンスデータ解放 |
| `FRDGAsyncScatterUploadBuffer` | `GPUScenePrivate.h` | 差分 GPU アップロード |
| `AddOrMergeInstanceRange()` | `GPUScene.cpp:2004` | クリア範囲を効率的にマージ |

---

## 関連リファレンス

- [[ref_gpuscene_class]] — FGPUScene メンバ変数・メソッド一覧
- [[ref_gpuscene_params]] — FGPUSceneResourceParameters シェーダーバインド定義
