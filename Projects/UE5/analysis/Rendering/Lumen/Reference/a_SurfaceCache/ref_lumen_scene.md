# リファレンス：LumenScene.cpp

- グループ: a - Surface Cache
- 上位: [[a_lumen_surface_cache]]
- 関連: [[ref_lumen_scene_data]] | [[ref_lumen_mesh_cards]] | [[ref_lumen_scene_gpu_driven_update]]
- ソース: `Engine/Source/Runtime/Renderer/Private/Lumen/LumenScene.cpp`

---

## 概要

Lumen シーンの**プリミティブ追加・削除・更新の CPU 側オーケストレーション**を担うファイル。  
`FScene` 内の `FLumenSceneData` を毎フレーム最新状態に保ち、  
GPU バッファ（CardBuffer・MeshCardsBuffer など）へのアップロードを管理する。

---

## 主要 CVar

| CVar | デフォルト | 説明 |
|------|----------|------|
| `r.LumenScene.GlobalSDF.Resolution` | 252 | Global SDF の解像度（Lumen 有効時）|
| `r.LumenScene.GlobalSDF.ClipmapExtent` | 2500.0 | 最初のクリップマップの半径（cm）|
| `r.LumenScene.FarField` | 0 | 遠距離フィールド有効/無効（変更時プロキシ再作成）|
| `r.LumenScene.FarField.OcclusionOnly` | 0 | 遠距離をオクルージョンのみにして高速化 |
| `r.LumenScene.FarField.MaxTraceDistance` | 1.0e6 | 遠距離の最大トレース距離（cm）|
| `r.LumenScene.FarField.FarFieldDitherScale` | 200.0 | 近距離/遠距離の境界ディザ幅（cm）|
| `r.LumenScene.SurfaceCache.CardCapturesPerFrame` | 300 | 1 フレームあたり最大キャプチャ Card 数 |
| `r.LumenScene.SurfaceCache.CardCaptureFactor` | 64 | キャプチャテクセル上限係数 |
| `r.LumenScene.SurfaceCache.CardCaptureRefreshFraction` | 0.125 | 既存ページ再キャプチャに使えるバジェット割合 |
| `r.LumenScene.SurfaceCache.Freeze` | 0 | 更新凍結（デバッグ用）|
| `r.LumenScene.SurfaceCache.Reset` | 0 | 全リセット（デバッグ用）|
| `r.LumenScene.SurfaceCache.AtlasSize` | 4096 | アトラスサイズ（px）|
| `r.LumenScene.ParallelUpdate` | 1 | 並列タスクで更新するか |
| `r.LumenScene.PrimitivesPerTask` | 128 | タスクあたりのプリミティブ数 |
| `r.LumenScene.MeshCardsPerTask` | 128 | タスクあたりの MeshCards 数 |
| `r.LumenScene.FastCameraMode` | 0 | 高速カメラ移動時の低品質モード |
| `r.LumenScene.UpdateViewOrigin` | 1 | View Origin の更新有効/無効 |

---

## フレームごとの更新フロー

```
FDeferredShadingRenderer::UpdateLumenScene(GraphBuilder)
  │
  ├─ 1. AllocateCardAtlases()      ← アトラス RDG テクスチャ生成（bReallocateAtlas 時のみ再生成）
  ├─ 2. UpdateLumenScenePrimitives() ← プリミティブ追加/削除/更新の CPU 処理
  ├─ 3. LumenScene::GPUDrivenUpdate() ← GPU Driven 可視性判定（オプション）
  ├─ 4. UpdateSurfaceCacheAllocationState() ← ページ解像度・割り当て更新
  ├─ 5. BuildCardUpdateList()       ← 今フレームでキャプチャする Card を選定
  ├─ 6. UpdateCardSceneBuffer()     ← GPU バッファへスキャッタアップロード
  └─ 7. LumenCardRenderer::CaptureCards() ← Card をアトラスにラスタライズ
```

---

## Lumen::UseFarField

```cpp
namespace Lumen {
    bool UseFarField(const FSceneViewFamily& ViewFamily);
}
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `ViewFamily` | `const FSceneViewFamily&` | EngineShowFlags と ShaderPlatform の参照用 |

### 戻り値
`bool` — 遠距離フィールドを使うべきなら true

### 使用箇所
- [[ref_lumen_scene_data]] `FLumenSceneData::AddMeshCards()` — 遠距離 MeshCards を追加するか判断
- [[ref_lumen_tracing_utils]] トレーシング設定で遠距離フィールド有効化の判定に使用
- [[ref_lumen_scene_gpu_driven_update]] `GPUDrivenUpdate()` — 遠距離グループを対象に含めるか

### 内部動作
```cpp
bool Lumen::UseFarField(const FSceneViewFamily& ViewFamily)
{
    return CVarLumenFarField.GetValueOnRenderThread() != 0
        && ViewFamily.EngineShowFlags.LumenFarFieldTraces
        && DoesPlatformSupportLumenGI(ViewFamily.GetShaderPlatform());
}
```

---

## UpdateLumenScenePrimitives

```cpp
void UpdateLumenScenePrimitives(FRHIGPUMask GPUMask, FScene* Scene);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GPUMask` | `FRHIGPUMask` | マルチ GPU 環境でのターゲット GPU マスク |
| `Scene` | `FScene*` | `FLumenSceneData` を保持するシーン |

### 使用箇所
- [[ref_lumen_scene]] `FDeferredShadingRenderer::UpdateLumenScene()` — 毎フレーム Lumen シーン更新の冒頭で呼ばれる

### 内部処理フロー

1. **削除フェーズ**（`PendingRemoveOperations` を処理）
   ```cpp
   for (const FLumenPrimitiveGroupRemoveInfo& RemoveInfo : LumenData.PendingRemoveOperations) {
       FLumenPrimitiveGroup& PrimitiveGroup = LumenData.PrimitiveGroups[RemoveInfo.PrimitiveGroupIndex];
       // プリミティブをグループから除去
       PrimitiveGroup.Primitives.RemoveSwap(RemoveInfo.Primitive);
       if (PrimitiveGroup.Primitives.IsEmpty()) {
           // グループが空になったら MeshCards と PrimitiveGroup を削除
           LumenData.RemoveMeshCards(RemoveInfo.PrimitiveGroupIndex);
           LumenData.PrimitiveGroups.RemoveSpan(RemoveInfo.PrimitiveGroupIndex, 1);
       }
   }
   LumenData.PendingRemoveOperations.Reset();
   ```

2. **追加フェーズ**（`PendingAddOperations` を処理）
   ```cpp
   for (FPrimitiveSceneInfo* PrimitiveSceneInfo : LumenData.PendingAddOperations) {
       // プリミティブが Lumen の対象か判定
       if (!ShouldTrackPrimitive(*PrimitiveSceneInfo)) continue;

       // Ray Tracing グループへのマージを試みる
       bool bMerged = TryMergeIntoRayTracingGroup(PrimitiveSceneInfo, LumenData);

       if (!bMerged) {
           // 新規 PrimitiveGroup を作成
           int32 PrimitiveGroupIndex = LumenData.PrimitiveGroups.AddSpan(1);
           FLumenPrimitiveGroup& NewGroup = LumenData.PrimitiveGroups[PrimitiveGroupIndex];
           NewGroup.Primitives.Add(PrimitiveSceneInfo);
           NewGroup.CardResolutionScale = ...;
           // MeshCards を生成
           LumenData.AddMeshCards(PrimitiveGroupIndex);
       }
   }
   LumenData.PendingAddOperations.Reset();
   ```

3. **更新フェーズ**（`PendingUpdateOperations` を処理）
   ```cpp
   for (FPrimitiveSceneInfo* PrimitiveSceneInfo : LumenData.PendingUpdateOperations) {
       int32 MeshCardsIndex = GetMeshCardsIndexForPrimitive(PrimitiveSceneInfo, LumenData);
       if (MeshCardsIndex != -1) {
           LumenData.UpdateMeshCards(
               PrimitiveSceneInfo->Proxy->GetLocalToWorld(),
               MeshCardsIndex,
               *PrimitiveSceneInfo->Proxy->GetMeshCardsBuildData());
       }
   }
   LumenData.PendingUpdateOperations.Reset();
   ```

---

## FDeferredShadingRenderer::UpdateLumenScene

```cpp
void FDeferredShadingRenderer::UpdateLumenScene(FRDGBuilder& GraphBuilder);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `GraphBuilder` | `FRDGBuilder&` | RDG ビルダー（テクスチャ・バッファ生成とパス追加）|

### 使用箇所
- `FDeferredShadingRenderer::Render()` — GBuffer パス前に呼ばれる

### 内部処理フロー

1. **アトラスの生成/確認**
   ```cpp
   LumenData.AllocateCardAtlases(GraphBuilder, LumenFrameTemporaries, &ViewFamily);
   ```

2. **ViewOrigin の更新**
   ```cpp
   for (const FViewInfo& View : Views) {
       FLumenViewOrigin ViewOrigin;
       ViewOrigin.Init(View);
       LumenFrameTemporaries.ViewOrigins.Add(ViewOrigin);
   }
   ```

3. **プリミティブの追加/削除/更新**
   ```cpp
   UpdateLumenScenePrimitives(GPUMask, Scene);
   ```

4. **GPU Driven 更新（オプション）**
   ```cpp
   if (CVarLumenGPUDrivenUpdate.GetValueOnRenderThread()) {
       LumenScene::GPUDrivenUpdate(GraphBuilder, Scene, Views, LumenFrameTemporaries);
   }
   ```

5. **Surface Cache アロケーション更新**
   ```cpp
   UpdateSurfaceCacheAllocationState(LumenData, Views, ...);
   BuildCardUpdateList(LumenData, CardCaptureRequests, ...);
   ```

6. **GPU バッファへのアップロード**
   ```cpp
   Lumen::UpdateCardSceneBuffer(GraphBuilder, UploadBuilder, LumenFrameTemporaries, ViewFamily, Scene);
   ```

7. **Card キャプチャ**（Surface Cache アトラスへのラスタライズ）
   ```cpp
   LumenCardRenderer.CaptureCards(GraphBuilder, Scene, Views, LumenFrameTemporaries, CardCaptureRequests);
   ```

---

<details>
<summary>UpdateSurfaceCacheAllocationState — ページ解像度・割り当て更新</summary>

```cpp
void UpdateSurfaceCacheAllocationState(
    FLumenSceneData& LumenData,
    const TArray<FViewInfo>& Views,
    FRHIGPUMask GPUMask,
    TArray<FSurfaceCacheRequest, SceneRenderingAllocator>& SurfaceCacheRequests);
```

### パラメータ
| 引数 | 型 | 説明 |
|------|-----|------|
| `LumenData` | `FLumenSceneData&` | Lumen シーンデータ（Card の解像度を更新する）|
| `Views` | `const TArray<FViewInfo>&` | ビュー一覧（カメラ距離計算に使用）|
| `GPUMask` | `FRHIGPUMask` | GPU マスク |
| `SurfaceCacheRequests` | `TArray<FSurfaceCacheRequest>&` | 出力: キャプチャが必要な Card のリクエストを積む |

### 内部処理フロー（概略）

1. **フィードバックバッファの読み取り** — GPU から「よく参照されるページ」を取得
2. **解像度の決定** — カメラ距離 → `DesiredLockedResLevel` を計算、空き容量チェック
3. **仮想ページの確保/解放** — `ReallocVirtualSurface()` / `FreeVirtualSurface()`
4. **LRU eviction** — アトラスが満杯の場合 `EvictOldestAllocation()` で古いページを解放
5. **キャプチャリクエストの生成** — 新規・再キャプチャが必要なページを `SurfaceCacheRequests` に追加

</details>

<details>
<summary>BuildCardUpdateList — 更新対象 Card の選定</summary>

```cpp
void BuildCardUpdateList(
    FLumenSceneData& LumenData,
    const TArray<FViewInfo>& Views,
    TArray<FSurfaceCacheRequest, SceneRenderingAllocator>& SurfaceCacheRequests,
    FLumenCardRenderer& LumenCardRenderer);
```

### 内部動作（概略）

1. `SurfaceCacheRequests` を優先度順にソート（新規ページ優先、次に距離近い順）
2. `r.LumenScene.SurfaceCache.CardCapturesPerFrame`（デフォルト 300）でリクエスト数を制限
3. テクセル総数制限（`AtlasTexels / CardCaptureFactor`）も適用
4. 制限内に収まる最終リストを `LumenCardRenderer` に渡す

</details>

<details>
<summary>LumenScene 名前空間 — LumenSceneGPUDrivenUpdate.h より</summary>

```cpp
namespace LumenScene {
    float GetCardMaxDistance(const FViewInfo& View);
    float GetCardTexelDensity();
    float GetFarFieldCardTexelDensity();
    float GetFarFieldCardMaxDistance();
    int32 GetCardMinResolution(bool bOrthographicCamera);
}
```

| 関数 | 説明 |
|------|------|
| `GetCardMaxDistance(View)` | カメラ FOV と設定から Card キャプチャの最大距離を返す |
| `GetCardTexelDensity()` | テクセル/世界単位の密度（近距離 Card 解像度の基準）|
| `GetFarFieldCardTexelDensity()` | 遠距離フィールド Card のテクセル密度 |
| `GetFarFieldCardMaxDistance()` | 遠距離フィールド Card の最大距離（`r.LumenScene.FarField.MaxTraceDistance`）|
| `GetCardMinResolution(bOrtho)` | Card の最小解像度（正投影カメラは異なる値）|

</details>

---

## プリミティブ追加の条件（ShouldTrackPrimitive）

以下の条件がすべて満たされる場合に Lumen がプリミティブを追跡する：

```
1. Proxy->AffectsDynamicIndirectLighting() == true
2. Proxy->GetMeshCardsBuildData() != nullptr
   （FPrimitiveSceneProxy が Lumen Card データを提供している）
3. 半透明メッシュでない（bOpaqueOrMasked == true）
4. r.Lumen.Scene が有効なプラットフォーム
5. 遠距離フィールド: Lumen::UseFarField() が true かつ bFarField フラグが設定されている
```
